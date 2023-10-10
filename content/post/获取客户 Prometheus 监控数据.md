---
title: 获取客户 Prometheus 监控数据
slug: get-customer-Prometheus-monitoring-data
tags: [ Python, Prometheus ]
date: 2023-09-08T14:18:08+08:00
---

## 业务背景

在排查问题时，想通过Grafana看板查看用户的监控，只能靠拍照，效率低，质量一般，设计一个方案能够方便的将问题出现前24小时的监控数据拿到，在本地导入，就能够在本地Grafana方便的查看。Prometheus 本身只提供了API查询的功能并没有导出数据功能，自带的Promtool也只提供验证规则文件和配置文件，调试等。

## 参考文章

[Analyzing Prometheus data with external tools](https://valyala.medium.com/analyzing-prometheus-data-with-external-tools-5f3e5e147639)

[Prometheus backfilling](https://medium.com/tlvince/prometheus-backfilling-a92573eb712c)

## **方案一：使用API导出转换成CSV**

使用API查询，将查询到的数据转换成CSV，刚好Grafana有插件能够将[CSV作为数据源](https://grafana.github.io/grafana-csv-datasource/)，经过实验后并不是特别顺利，能够读取到CSV，但没有成功绘制出图像。

## **总结**

经过实验后并不是特别顺利，能够读取到CSV但没有成功绘制出图像，看板中部分查询语句中包含看板变量，CSV数据源无法实现看板变量。

## **方案二：拷贝Prometheus数据文件**

Prometheus 按照两个小时为一个时间窗口，将两小时内产生的数据存储在一个块（Block）中。每个块都是一个单独的目录，里面含该时间窗口内的所有样本数据（chunks），元数据文件（meta.json）以及索引文件（index）。其中索引文件会将指标名称和标签索引到样板数据的时间序列中。此期间如果通过 API 删除时间序列，删除记录会保存在单独的逻辑文件 tombstone 当中。

Prometheus 为了防止丢失暂存在内存中的还未被写入磁盘的监控数据，引入了WAL机制。WAL被分割成默认大小为128M的文件段（segment），之前版本默认大小是256M，文件段以数字命名，长度为8位的整形。WAL的写入单位是页（page），每页的大小为32KB，所以每个段大小必须是页的大小的整数倍。如果WAL一次性写入的页数超过一个段的空闲页数，就会创建一个新的文件段来保存这些页，从而确保一次性写入的页不会跨段存储。这些数据暂时没有持久化，TSDB通过WAL将数据保存到磁盘上(保存的数据没有压缩，占用内存较大)，当出现宕机，启动多协程读取WAL，恢复数据。

```
[mingming.chen@m162p65 data]$ tree
.
├── 01E2MA5GDWMP69GVBVY1W5AF1X
│   ├── chunks               # 保存压缩后的时序数据，每个chunks大小为512M，超过会生成新的chunks
│   │   └── 000001
│   ├── index                # chunks中的偏移位置
│   ├── meta.json            # 记录block块元信息，比如 样本的起始时间、chunks数量和数据量大小等
│   └── tombstones           # 通过API方式对数据进行软删除,将删除记录存储在此处（API的删除方式，并不是立即将数据从chunks文件中移除）
├── 01E2MH175FV0JFB7EGCRZCX8NF
│   ├── chunks
│   │   └── 000001
│   ├── index
│   ├── meta.json
│   └── tombstones
├── 01E2MQWYDFQAXXPB3M1HK6T20A
│   ├── chunks
│   │   └── 000001
│   ├── index
│   ├── meta.json
│   └── tombstones
├── lock
├── queries.active
└── wal                      #防止数据丢失(数据收集上来暂时是存放在内存中,wal记录了这些信息)
    ├── 00000366             #每个数据段最大为128M，存储默认存储两个小时的数据量。
    ├── 00000367
    ├── 00000368
    ├── 00000369
    └── checkpoint.000365
        └── 00000000
```

无论是block数据还是wal数据，都是可以直接打包，转移到本地的Prometheus，需要注意的是版本问题和本地Prometheus不能有数据，如果本地监控数据目录不为空，那么导入时会出现问题（因为时间问题），只需要近期数据，太远的数据没有价值，可以通过block文件里面的meta.json查看时间戳
例如，在meta.json中，会看到这样的内容：

```
jsonCopy code{
  "ulid": "01E2MA5GDWMP69GVBVY1W5AF1X",
  "minTime": 1609459200000,
  "maxTime": 1609545600000,
}
```

[tsdb创建快照](https://www.robustperception.io/taking-snapshots-of-prometheus-data/)的方式，需要Prometheus 在启动时开启了--web.enable-admin-api，发送POST请求后就会在Prometheus 数据目录创建block文件的硬链接，作为快照。可以传输至其他机器恢复。
复制数据目录和tsdb快照都能够复制数据迁移到另一台Prometheus ，区别在于下表

| 对比     | 好处                                       | 坏处                               |
| -------- | ------------------------------------------ | ---------------------------------- |
| file     | 能够灵活的决定是否拷贝这个block文件        | 版本兼容性可能一般                 |
| snapshot | 操作简单不易出错，版本支持会比操作文件更好 | 只能对整个数据库快照，不能选择范围 |

## **总结**

能够将客户的监控数据转移到我们的Prometheus （数据量不太可控），Grafana照常使用不影响，不会影响看板

## **方案三：tsdb command-line tool /promtool tsdb dump**

这里有一个TSDB的[PR](https://github.com/prometheus-junkyard/tsdb/pull/532)，让tsdb支持导出功能，但是随着Prometheus 和TSDB的开发，TSDB已经被合入了Prometheus ，Prometheus 的[官方文档](https://prometheus.io/docs/prometheus/latest/command-line/promtool/#promtool-tsdb-dump)中记录的promtool tsdb dump提供了导出tsdb数据的功能

promtool tsdb dump 的使用方法见官方文档，它会读取Prometheus 数据目录的文件，作为stdout打印出来，我们可以重定向到文本文件再使用grep之类的工具进行过滤（比如，我们只需要和es相关的指标序列），拿到了想要的数据，想办法导入到我们的Prometheus 。

使用promtool tsdb dump导出的数据并不是Openmetrics格式。实际上，它是一种更为底层、直接反映存储结构的格式，主要用于调试和低级数据操作。

Prometheus [2.24.0 / 2021-01-06](https://github.com/prometheus/prometheus/blob/main/CHANGELOG.md#2240--2021-01-06)支持了[backfilling](https://prometheus.io/docs/prometheus/latest/storage/#backfilling-from-openmetrics-format)功能（能够将Openmetric格式的数据导入到Prometheus ）

于是有人[提议](https://github.com/prometheus/prometheus/issues/8280)promtool tsdb dump导出的数据如果是Openmetric格式，那么可以实现更方便的导入和导出，Prometheus 作者的[意见](https://github.com/prometheus/prometheus/issues/8280#issuecomment-743888219)，迁移数据的正确方式就是tsdb的快照功能。

## **总结**

如果使用promtool tsdb dump导出，然后通过shell/python转换成Openmetric标准格式文本文件，就能够通过我们的promtool导入，测试已经导入成功。问题是promtool tsdb dump此命令不能对数据进行筛选，会导致无意义的耗时。

## **方案四：使用API查询指定时间段指标，转换成Openmetric**

Prometheus [2.24.0 / 2021-01-06](https://github.com/prometheus/prometheus/blob/main/CHANGELOG.md#2240--2021-01-06)支持了[backfilling](https://prometheus.io/docs/prometheus/latest/storage/#backfilling-from-openmetrics-format)功能（能够将Openmetric格式的数据导入到Prometheus ）

通过API查询到想要的数据（能够自定义时间范围，自定义指标名称，自定义步长），没有多余的耗时



## 总结

以上代码顺利查询指标并导出,仅依赖request，如果有必要或许能够通过shell处理，能够进一步减少依赖，直接运行即可，导出200多条时间序列，前30小时的数据，最后文本文件的大小不到500M,在可以接受的范围，主要原因是Openmetric格式中太多的重复字段。

```python
# -*- coding: utf-8 -*-
# !/usr/bin/python
import datetime
import requests
import sys
import re

requests.packages.urllib3.disable_warnings()
"""
脚本会通过Prometheus的api查询指定集群的关于ElasticSearch的指标,并将结果保存到本地文件中
"""


def get_user_input(prompt):
    if sys.version_info.major == 2:
        # Python 2
        return raw_input(prompt)
    elif sys.version_info.major == 3:
        # Python 3
        return input(prompt)
    else:
        raise Exception("不支持的Python版本")


prometheus_url = "http://prometheus.cn-zhangjiakou-zsearch2.elasticsearch.aliyuncs.com"
username = "prometheus"
password = "AliOS%1688"
auth = None
if username.strip() and password.strip():
    auth = (username, password)
elif username.strip() or password.strip():
    print("请提供完整的用户名和密码或者都不提供以跳过认证。")
    sys.exit(1)

response = requests.get("{}/api/v1/label/cluster/values".format(prometheus_url), auth=auth, verify=False)
if response.status_code == 200:
    cluster_list = response.json()["data"]
    if len(cluster_list) == 0:
        raise ValueError("没有查询到集群")
    print("监控的集群有:")
    for cluster in cluster_list:
        print(cluster)
else:
    raise ValueError(
        "获取集群列表失败,请检查{}/api/v1/label/cluster/values,原因:{}".format(prometheus_url, response.reason))

cluster = get_user_input("请输入要查询集群名称:")

if cluster not in cluster_list:
    print("请检查输入！")
    sys.exit(1)

# 访问/api/v1/status/config获取Prometheus的默认scrape_interval
yaml_data = requests.get('{}/api/v1/status/config'.format(prometheus_url), auth=auth, verify=False).json()['data'][
    'yaml']
scrape_interval = re.search(r'scrape_interval:\s+(\d+)', yaml_data)
default_step = int(scrape_interval.group(1))  # 查询Prometheus配置的值作为默认的 step,避免重复数据

step = get_user_input("当前配置的采集间隔为{}秒,建议默认或更大,避免重复数据,请输入采集间隔秒数:".format(default_step))
hours = get_user_input("将自动计算最大查询范围,建议指定所需较小的时间范围,请输入往前查的小时数:")


def is_positive_integer(parameters):
    """
判断输入的参数是否为正整数
"""

    try:
        num = int(parameters)
        if num > 0:
            return True
        else:
            return False
    except ValueError:
        return False


if step == "" and hours == "":
    step = default_step
    hours = int(11000 / (60 / int(step)) / 60)
    print("将查询前{}小时的数据,采集间隔为{}秒".format(hours, step))
elif step != "" and hours == "" and is_positive_integer(step):
    step = int(step)
    hours = int(11000 / (60 / int(step)) / 60)
    print("将查询前{}小时的数据,采集间隔为{}秒".format(hours, step))

elif step == "" and hours != "" and is_positive_integer(hours):
    step = default_step
    hours = int(hours)
    print("将查询前{}小时的数据,采集间隔为{}秒".format(hours, step))

elif step != "" and hours != "" and is_positive_integer(step) and is_positive_integer(hours):
    step = int(step)
    hours = int(hours)
    print("将查询前{}小时的数据,采集间隔为{}秒".format(hours, step))
else:
    print("请检查输入！")
    sys.exit(1)

end_time = datetime.datetime.now().strftime("%Y-%m-%dT%H:%M:%SZ")
query_time = datetime.datetime.now() - datetime.timedelta(hours=hours)
start_time = query_time.strftime("%Y-%m-%dT%H:%M:%SZ")

series = requests.get('{}/api/v1/label/__name__/values'.format(prometheus_url), auth=auth, verify=False)
metric_list = [metric for metric in series.json()['data'] if
               '{}'.format('elasticsearch') in metric]  # 将查询出所有带有elasticsearch的指标
promQL_list = ['{}{{cluster="{}"}}'.format(metric, cluster) for metric in metric_list]
print("本次一共查询了{}个指标".format(len(promQL_list)))
filename = 'openmetrics_{}_{}_{}.txt'.format(cluster, step, hours)
with open(filename, 'a') as f:
    for promQL in promQL_list:
        metrics = requests.get(
            '{}/api/v1/query_range?query={}&start={}&end={}&step={}s'.format(
                prometheus_url, promQL, start_time, end_time, step), auth=auth, verify=False)
        if metrics.status_code != 200:
            print("查询失败,状态码为{}".format(metrics.status_code))
            sys.exit(1)
        else:
            prometheus_data = metrics.json()
            for result in prometheus_data['data']['result']:
                metric_name = result['metric']['__name__']
                labels = []
                for key, value in result['metric'].items():
                    if key != '__name__':
                        labels.append('{}="{}"'.format(key, value))
                labels = ','.join(labels)
                openmetrics = []
                for value in result['values']:
                    openmetrics.append('{}{{{}}} {} {}\n'.format(metric_name, labels, value[1], value[0]))
                openmetrics = ''.join(openmetrics)
                f.write(openmetrics)
    f.write('# EOF\n')  # 文档末尾标志必须添加,promtool才能正常识别
```

目前可行性最高的方案，本地测试和测试环境测试均已成功导出导入，由于Openmetric重复率较高，有较高的压缩比例，4GB的文本文件,可以压缩为40MB,导入到本地的 Prometheus ，重启 Prometheus （让其重载块文件）在Prometheus 和Grafana均可正常查询。
