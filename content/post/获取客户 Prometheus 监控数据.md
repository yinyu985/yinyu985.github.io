---
title: 获取客户 Prometheus 监控数据
slug: get-customer-Prometheus-monitoring-data
tags: [ Python, Prometheus ]
date: 2023-09-08T14:18:08+08:00
---

## 业务背景

为了方便二线同事排查问题时方便查看Grafana监控，现在的查看方式是，网络不通只能靠客户拍照，模糊，效率低，为了实现将用户的Prometheus数据可控的迁移到我们本地，Prometheus本身只提供了API查询的功能并没有导出数据功能，自带的Promtool也只提供验证规则文件和配置文件，调试等，无法实现监控数据的导出。

## 参考文章

[Analyzing Prometheus data with external tools](https://valyala.medium.com/analyzing-prometheus-data-with-external-tools-5f3e5e147639)

[Prometheus backfilling](https://medium.com/tlvince/prometheus-backfilling-a92573eb712c)

## **方案一：使用API导出转换成CSV**

```python
def query_prometheus_and_export_to_dataframe(prometheus_url, query, start_time, end_time, step_time):
    """
    查询Prometheus并将结果导出为DataFrame
    Args:
        prometheus_url: Prometheus的URL
        query: PromQL查询表达式
        start_time: 查询开始时间
        end_time: 查询结束时间
        step: 查询步长
    Returns:
        DataFrame对象
    # 构造查询请求
    url = f"{prometheus_url}/api/v1/query_range?query={query}"
    if start_time is not None and end_time is not None:
        url += f"&start={start_time}&end={end_time}&step={step_time}"
    # 发送查询请求
    response = requests.get(url)
    if response.status_code != 200:
        raise Exception(f"查询Prometheus失败: {response.status_code}")
    # 解析查询结果
    result = response.json()
    if "data" not in result:
        raise Exception(f"查询结果无效: {result}")
    data = result["data"]["result"]
    return data
prometheus_url = "http://localhost:9090"
query = "node_load1{}"
start_time = "2023-09-04T00:00:00Z"
end_time = "2023-09-04T23:59:59Z"
step_time = "10s"
file_path = "prometheus_data.csv"
data = query_prometheus_and_export_to_dataframe(prometheus_url, query, start_time, end_time, step_time)
```

可以将查询到的数据转换成CSV，刚好Grafana有插件能够将[CSV作为数据源](https://grafana.github.io/grafana-csv-datasource/)。

## **总结**

经过实验后并不是特别顺利，能够读取到CSV但没有成功绘制出图像，当看板变多需要查询的指标和查询的PromQL都会变多，并且部分查询语句中包含看板变量，CSV数据源无法实现看板变量。

## **方案二：拷贝Prometheus数据文件**

Prometheus 按照两个小时为一个时间窗口，将两小时内产生的数据存储在一个块（Block）中。每个块都是一个单独的目录，里面含该时间窗口内的所有样本数据（chunks），元数据文件（meta.json）以及索引文件（index）。其中索引文件会将指标名称和标签索引到样板数据的时间序列中。此期间如果通过 API 删除时间序列，删除记录会保存在单独的逻辑文件 tombstone 当中。

Prometheus为了防止丢失暂存在内存中的还未被写入磁盘的监控数据，引入了WAL机制。WAL被分割成默认大小为128M的文件段（segment），之前版本默认大小是256M，文件段以数字命名，长度为8位的整形。WAL的写入单位是页（page），每页的大小为32KB，所以每个段大小必须是页的大小的整数倍。如果WAL一次性写入的页数超过一个段的空闲页数，就会创建一个新的文件段来保存这些页，从而确保一次性写入的页不会跨段存储。这些数据暂时没有持久化，TSDB通过WAL将数据保存到磁盘上(保存的数据没有压缩，占用内存较大)，当出现宕机，启动多协程读取WAL，恢复数据。

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

meta.json: 文件包含有关数据块的元信息，包括时间范围（minTime和maxTime），这是以Unix时间戳（毫秒）的形式给出的。您可以使用这些信息来确定哪些数据块包含您需要的24小时的数据。

例如，在meta.json中，您可能会看到这样的内容：

```
jsonCopy code{
  "ulid": "01E2MA5GDWMP69GVBVY1W5AF1X",
  "minTime": 1609459200000,
  "maxTime": 1609545600000,
  ...
}
```

[tsdb创建快照](https://www.robustperception.io/taking-snapshots-of-prometheus-data/)的方式，需要Prometheus在启动时开启了--web.enable-admin-api，发送POST请求后就会在Prometheus数据目录创建block文件的硬链接，作为快照。可以传输至其他机器恢复。复制数据目录和tsdb快照都能够复制数据迁移到另一台Prometheus，区别在于下表。

| 对比     | 好处                                       | 坏处                               |
| -------- | ------------------------------------------ | ---------------------------------- |
| file     | 能够灵活的决定是否拷贝这个block文件        | 版本兼容性可能一般                 |
| snapshot | 操作简单不易出错，版本支持会比操作文件更好 | 只能对整个数据库快照，不能选择范围 |

## **总结**

这样确实能够将客户的监控数据转移到我们的Prometheus，Grafana照常使用不影响，对于绘图没有任何影响，但是不能过滤，会传输无意义的数据,耗费时间。

## **方案三：tsdb command-line tool /promtool tsdb dump**

这里有一个TSDB的[PR](https://github.com/prometheus-junkyard/tsdb/pull/532)，让tsdb支持导出功能，但是随着Prometheus和TSDB的开发，TSDB已经被合入了Prometheus，[Prometheus](https://github.com/prometheus/prometheus/blob/main/CHANGELOG.md#2420--2023-01-31)的[官方文档](https://prometheus.io/docs/prometheus/latest/command-line/promtool/#promtool-tsdb-dump)中记录的`promtool tsdb dump`提供了导出tsdb数据的功能

`promtool tsdb dump`的使用方法见官方文档，它会读取Prometheus数据目录的文件，作为stdout打印出来，我们可以重定向到文本文件再使用grep之类的工具进行过滤（比如，我们只需要和es相关的指标序列），拿到了想要的数据，想办法导入到我们的Prometheus。

使用promtool tsdb dump导出的数据并不是OpenMetrics格式。实际上，它是一种更为底层、直接反映存储结构的格式，主要用于调试和低级数据操作。

Prometheus[2.24.0 / 2021-01-06](https://github.com/prometheus/prometheus/blob/main/CHANGELOG.md#2240--2021-01-06)支持了[backfilling](https://prometheus.io/docs/prometheus/latest/storage/#backfilling-from-openmetrics-format)功能（能够将openmetric格式的数据导入到Prometheus）

于是有人[提议](https://github.com/prometheus/prometheus/issues/8280)promtool tsdb dump导出的数据如果是openmetric格式，那么可以实现更方便的导入和导出，Prometheus作者的[意见](https://github.com/prometheus/prometheus/issues/8280#issuecomment-743888219)，迁移数据的正确方式就是tsdb的快照功能。

## **总结**

如果使用promtool tsdb dump导出，然后通过shell脚本转换成openmetric标准文本文件，就能够通过promtool导入，具体需要测试。

## **方案四：使用API查询指定时间段指标，转换成Openmetric**

查询代码和之前类似，但是这次就不需要CSV了，直接通过脚本转换成Openmetric。

```python
import datetime
import requests

"""
定义需要查询的Prometheus链接
定义需要查询的多少小时前的数据
定义需要查询的时间步长，步长越小，数据点越多，能够查询的时间范围就变小
（60/【步长】(单位秒)）*60 *【小时数】<11000
"""
prometheus_url = 'http://localhost:9090'
end_time = datetime.datetime.now().strftime("%Y-%m-%dT%H:%M:%SZ")
query_time = datetime.datetime.now() - datetime.timedelta(hours=30)
start_time = query_time.strftime("%Y-%m-%dT%H:%M:%SZ")
print(f"开始查询从{start_time}到{end_time}的指标")

series = requests.get('{}/api/v1/label/__name__/values'.format(prometheus_url))  # 查询全部的指标名称
print(f'Prometheus中有{len(series.json()["data"])}条指标')
queryQL_list = [queryQL for queryQL in series.json()['data'] if 'node' in queryQL] # 查询所有包含”node“的指标
print(f"本次一共查询了{len(queryQL_list)}条指标")

with open('openmetrics.txt', 'w') as f:
    for queryQL in queryQL_list:
        metric = requests.get(
            f'{prometheus_url}/api/v1/query_range?query={queryQL}&start={start_time}&end={end_time}&step=10s')  # 定义步长为10s
        if metric.json()['status'] != 'success':
            print(f"查询失败，请重新计算查询点是否超过了11000")
        else:
            prometheus_data = metric.json()
            for result in prometheus_data['data']['result']:
                metric_name = result['metric']['__name__']
                labels = []
                for key, value in result['metric'].items():
                    if key != '__name__':
                        labels.append(f'{key}="{value}"')
                labels = ','.join(labels)
                openmetrics = []

                for value in result['values']:
                    # print(f'{metric_name}{{{labels}}} {value[1]} {value[0]}\n')
                    openmetrics.append(f'{metric_name}{{{labels}}} {value[1]} {value[0]}\n')
                openmetrics = ''.join(openmetrics)
                f.write(openmetrics)
    f.write('# EOF')  # 文档末尾添加结尾标志
print("写入完成")
```

## 总结

以上代码顺利查询指标并导出,仅依赖request，如果有必要或许能够通过shell处理，能够进一步减少依赖，直接运行即可，导出200多条时间序列，前30小时的数据，最后文本文件的大小不到500M,在可以接受的范围，主要原因是Openmetric格式中太多的重复字段。
