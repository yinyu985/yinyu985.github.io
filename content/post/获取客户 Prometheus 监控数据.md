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

以上代码顺利查询指标并导出,仅依赖requests，如果有必要或许能够通过shell处理，能够进一步减少依赖，直接运行即可，导出200多条时间序列，前30小时的数据，最后文本文件的大小不到500M,在可以接受的范围，主要原因是Openmetric格式中太多的重复字段。

```python
# -*- coding: utf-8 -*-
# !/usr/bin/python
import subprocess
import datetime
import shutil
import json
import sys
import re
import os

if sys.version_info.major == 2:
    from urllib import quote
else:
    from urllib.parse import quote


def get_user_input(prompt):
    return raw_input(prompt) if sys.version_info.major == 2 else input(prompt)


def get_input_or_default(prompt, default_value):
    while True:
        user_input = get_user_input(prompt).strip()
        if user_input.isdigit():
            return int(user_input)
        elif user_input == '':
            return default_value
        else:
            print("请输入一个正整数或者直接回车使用默认值！")


def get_data_from_prometheus(url, auth, headers=None):
    try:
        try:
            import requests
            from requests.packages.urllib3.exceptions import InsecureRequestWarning
            requests.packages.urllib3.disable_warnings(InsecureRequestWarning)
            response = requests.get(url, auth=auth, verify=False, headers=headers)
            response.raise_for_status()
            print("request请求的链接是,{}".format(response.url))
            data = response.json()
        except ImportError:
            cmd = ['curl', '-s', '-k', '-L', '--keepalive', '--compressed']
            if auth is not None:
                cmd += ['-u', '{}:{}'.format(auth[0], auth[1])]
            if headers is not None:
                for key, value in headers.items():
                    cmd += ['-H', '{}: {}'.format(key, value)]
            cmd.append(url)
            print("curl请求的链接,{}".format(subprocess.list2cmdline(cmd)))
            response = subprocess.check_output(cmd)
            data = json.loads(response)

        if 'data' in data and 'result' in data['data']:
            result_list = data['data']['result']
            if not result_list:
                print("该指标为空，跳过")
        return data
    except Exception as e:
        print("获取数据失败,请检查{},原因:{}".format(url, str(e)))
        sys.exit(1)


intro_message = """
################################################################
脚本将通过Prometheus内置的API接口导出ElasticSearch相关的监控数据
文本压缩率很高，当导出文本文件超过100MB将自动压缩，将压缩包传出即可
Prometheus硬性要求单次查询（3600/查询间隔）* 查询小时数 ≤ 11000
################################################################
"""
print(intro_message)

while True:
    prometheus_url = get_user_input("请输入要查询的Prometheus地址:")
    """判断开头是否是http或者https"""
    if prometheus_url.startswith("http://") or prometheus_url.startswith("https://"):
        break
    else:
        print("请输入正确的Prometheus地址，以http或者https开头")
        continue
username = get_user_input("请输入要查询的prometheus用户名/无认证可回车跳过:")
password = get_user_input("请输入要查询的prometheus密码/无认证可回车跳过:")

auth = (username, password) if username.strip() and password.strip() else None
if auth is None and (username.strip() or password.strip()):
    print("请提供完整的用户名和密码/无认证可回车跳过。")
    sys.exit(1)

cluster_list_url = "{}/api/v1/label/cluster/values".format(prometheus_url)
cluster_list = get_data_from_prometheus(cluster_list_url, auth)["data"]

if not cluster_list:
    print("没有查询到集群")
    sys.exit(1)

print("监控的集群有:")
for cluster in cluster_list:
    print(cluster)

while True:
    cluster = get_user_input("请输入要查询集群名称:")
    if cluster not in cluster_list:
        print("请输入正确的集群名称")
    else:
        break

config_url = '{}/api/v1/status/config'.format(prometheus_url)
config = get_data_from_prometheus(config_url, auth)['data']['yaml']
scrape_interval = re.search(r'scrape_interval:\s+(\d+)', config)
default_step = int(scrape_interval.group(1))

step = get_input_or_default(
    "当前配置的采集间隔为{}秒,回车使用默认值或输入更大值,避免重复数据:".format(default_step), default_step)


def get_time_input(prompt, step):
    while True:
        time_str = get_user_input(prompt)
        if re.match(r'\d{8}-\d{8}', time_str):
            start_date, end_date = time_str[:8], time_str[9:]
            try:
                start_time = datetime.datetime.strptime(start_date, "%Y%m%d").strftime("%Y-%m-%dT%H:%M:%SZ")
                end_time = datetime.datetime.strptime(end_date, "%Y%m%d").strftime("%Y-%m-%dT%H:%M:%SZ")
                hours = int((datetime.datetime.strptime(end_date, "%Y%m%d") -
                             datetime.datetime.strptime(start_date, "%Y%m%d")).total_seconds() / 3600)
                return start_time, end_time, hours
            except ValueError:
                print("请输入正确的日期格式，例如：20231111-20231112")
        elif time_str.isdigit():
            hours = int(time_str)
            if hours * 60 * 60 / step > 11000:
                print("查询数据量超过11000，请重新输入")
            else:
                end_time = datetime.datetime.now().strftime("%Y-%m-%dT%H:%M:%SZ")
                query_time = datetime.datetime.now() - datetime.timedelta(hours=hours)
                start_time = query_time.strftime("%Y-%m-%dT%H:%M:%SZ")
                return start_time, end_time, hours
        else:
            print("请输入正确的时间范围，例如：20231111-20231112 或者 输入小时数")


start_time, end_time, hours = get_time_input("请输入查询时间范围（例如：20231111-20231112）或者输入小时数：", step)
print("将查询从{}到{}的数据，采集间隔为{}秒".format(start_time, end_time, step))

series_url = '{}/api/v1/label/__name__/values'.format(prometheus_url)
series = get_data_from_prometheus(series_url, auth)
metric_list = [metric for metric in series['data'] if
               'elasticsearch' in metric and 'index' not in metric and 'indices' not in metric]
promQL_list = ['{}{{pod=~"{}.*"}}'.format(metric, cluster) for metric in metric_list]

if len(promQL_list) == 0:
    print("没有找到ElasticSearch相关的指标")
    sys.exit(1)
print("本次查询共涉及{}个指标".format(len(promQL_list)))
filename = 'openmetrics_{}_{}_{}.txt'.format(cluster, step, hours)
query_start_time = datetime.datetime.now()
with open(filename, 'a') as f:
    for promQL in promQL_list:
        promQL = quote(promQL)
        metrics_url = '{}/api/v1/query_range?query={}&start={}&end={}&step={}s'.format(prometheus_url, promQL,
                                                                                       start_time, end_time, step)
        headers = {'Content-Type': 'application/json'}
        prometheus_data = get_data_from_prometheus(metrics_url, auth, headers)
        for result in prometheus_data['data']['result']:
            metric_name = result['metric']['__name__']
            labels = []
            for key, value in result['metric'].items():
                if key != '__name__':
                    labels.append('{}="{}"'.format(key, value))
            labels = ','.join(labels)
            metrics_data = []
            for value in result['values']:
                metrics_data.append('{}{{{}}} {} {}\n'.format(metric_name, labels, value[1], value[0]))
            f.write(''.join(metrics_data))
    f.write('# EOF\n')  # 文档末尾标志必须添加,promtool才能正常识别

file_size = os.path.getsize(filename)  # 返回的是字节数

if file_size > 100 * 1024 * 1024:
    shutil.make_archive(filename, 'zip', '.', filename)
query_end_time = datetime.datetime.now()
elapsed_time = query_end_time - query_start_time
elapsed_seconds = elapsed_time.total_seconds()
hours, remainder = divmod(elapsed_seconds, 3600)
minutes, seconds = divmod(remainder, 60)
print("查询总耗时: {:02}:{:02}:{:02}".format(int(hours), int(minutes), int(seconds)))
```

目前可行性最高的方案，本地测试和测试环境测试均已成功导出导入，由于Openmetric重复率较高，有较高的压缩比例，4GB的文本文件,可以压缩为40MB,导入到本地的 Prometheus ，重启 Prometheus （让其重载块文件）在Prometheus 和Grafana均可正常查询。

## 后续更新

新增了一种请求方式，即使没有requests，也可以通过curl来发送请求，没有任何的外部依赖（经过配置和测试curl没有明显落后）

新增了错误重输的功能，当用户输入不满足要求时，会提示重新输入并不会脚本退出

新增了指定时间范围查询，假如故障发生在上周，想往前查多少小时（原来接受输入的方式已经无法满足要求）
