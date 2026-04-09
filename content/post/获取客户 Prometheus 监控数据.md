---
title: 获取客户 Prometheus 监控数据
slug: get-customer-Prometheus-monitoring-data
tags: [Python, Prometheus]
date: 2023-09-08T14:18:08+08:00
---

## 业务背景

在排查问题时，想通过 Grafana 看板查看用户的监控，只能靠拍照，效率低，质量一般。设计一个方案能够方便地将问题出现前 24 小时的监控数据拿到，在本地导入，就能够在本地 Grafana 方便地查看。Prometheus 本身只提供了 API 查询的功能，并没有导出数据功能；自带的 `promtool` 也只提供验证规则文件和配置文件、调试等功能。

## 参考文章

- [Analyzing Prometheus data with external tools](https://valyala.medium.com/analyzing-prometheus-data-with-external-tools-5f3e5e147639)
- [Prometheus backfilling](https://medium.com/tlvince/prometheus-backfilling-a92573eb712c)

## 方案一：使用 API 导出转换成 CSV

使用 API 查询，将查询到的数据转换成 CSV。刚好 Grafana 有插件能够将 [CSV 作为数据源](https://grafana.github.io/grafana-csv-datasource/)。经过实验后并不是特别顺利，能够读取到 CSV，但没有成功绘制出图像。

## 总结

经过实验后并不是特别顺利，能够读取到 CSV 但没有成功绘制出图像。看板中部分查询语句中包含看板变量，CSV 数据源无法实现看板变量。

## 方案二：拷贝 Prometheus 数据文件

Prometheus 按照两个小时为一个时间窗口，将两小时内产生的数据存储在一个块（Block）中。每个块都是一个单独的目录，里面包含该时间窗口内的所有样本数据（`chunks`）、元数据文件（`meta.json`）以及索引文件（`index`）。其中索引文件会将指标名称和标签索引到样本数据的时间序列中。此期间如果通过 API 删除时间序列，删除记录会保存在单独的逻辑文件 `tombstone` 当中。

Prometheus 为了防止丢失暂存在内存中的还未被写入磁盘的监控数据，引入了 WAL 机制。WAL 被分割成默认大小为 128M 的文件段（segment），之前版本默认大小是 256M，文件段以数字命名，长度为 8 位的整型。WAL 的写入单位是页（page），每页的大小为 32KB，所以每个段大小必须是页的大小的整数倍。如果 WAL 一次性写入的页数超过一个段的空闲页数，就会创建一个新的文件段来保存这些页，从而确保一次性写入的页不会跨段存储。这些数据暂时没有持久化，TSDB 通过 WAL 将数据保存到磁盘上（保存的数据没有压缩，占用内存较大），当出现宕机时，启动多协程读取 WAL，恢复数据。

```
[mingming.chen@m162p65 data]$ tree
.
├── 01E2MA5GDWMP69GVBVY1W5AF1X
│   ├── chunks               # 保存压缩后的时序数据，每个 chunks 大小为 512M，超过会生成新的 chunks
│   │   └── 000001
│   ├── index                # chunks 中的偏移位置
│   ├── meta.json            # 记录 block 块元信息，比如样本的起始时间、chunks 数量和数据量大小等
│   └── tombstones           # 通过 API 方式对数据进行软删除，将删除记录存储在此处（API 的删除方式，并不是立即将数据从 chunks 文件中移除）
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
└── wal                      # 防止数据丢失（数据收集上来暂时是存放在内存中，wal 记录了这些信息）
    ├── 00000366             # 每个数据段最大为 128M，存储默认存储两个小时的数据量
    ├── 00000367
    ├── 00000368
    ├── 00000369
    └── checkpoint.000365
        └── 00000000
```

无论是 block 数据还是 wal 数据，都是可以直接打包，转移到本地的 Prometheus。需要注意的是版本问题，且本地 Prometheus 不能有数据。如果本地监控数据目录不为空，那么导入时会出现问题（因为时间问题）。只需要近期数据，太远的数据没有价值，可以通过 block 文件里面的 `meta.json` 查看时间戳。

例如，在 `meta.json` 中，会看到这样的内容：

```json
{
  "ulid": "01E2MA5GDWMP69GVBVY1W5AF1X",
  "minTime": 1609459200000,
  "maxTime": 1609545600000
}
```

[tsdb 创建快照](https://www.robustperception.io/taking-snapshots-of-prometheus-data/) 的方式，需要 Prometheus 在启动时开启了 `--web.enable-admin-api`，发送 POST 请求后就会在 Prometheus 数据目录创建 block 文件的硬链接，作为快照。可以传输至其他机器恢复。

复制数据目录和 tsdb 快照都能够复制数据迁移到另一台 Prometheus，区别在于下表：

| 对比     | 好处                                       | 坏处                               |
| -------- | ------------------------------------------ | ---------------------------------- |
| file     | 能够灵活地决定是否拷贝这个 block 文件      | 版本兼容性可能一般                 |
| snapshot | 操作简单不易出错，版本支持会比操作文件更好 | 只能对整个数据库快照，不能选择范围 |

## 总结

能够将客户的监控数据转移到我们的 Prometheus（数据量不太可控），Grafana 照常使用不影响，不会影响看板。

## 方案三：tsdb command-line tool / promtool tsdb dump

这里有一个 TSDB 的 [PR](https://github.com/prometheus-junkyard/tsdb/pull/532)，让 tsdb 支持导出功能。但随着 Prometheus 和 TSDB 的开发，TSDB 已被合入 Prometheus。Prometheus 的 [官方文档](https://prometheus.io/docs/prometheus/latest/command-line/promtool/#promtool-tsdb-dump) 中记录的 `promtool tsdb dump` 提供了导出 tsdb 数据的功能。

`promtool tsdb dump` 的使用方法见官方文档，它会读取 Prometheus 数据目录的文件，作为 stdout 打印出来。我们可以重定向到文本文件，再使用 `grep` 之类的工具进行过滤（比如，我们只需要和 es 相关的指标序列）。拿到了想要的数据，想办法导入到我们的 Prometheus。

使用 `promtool tsdb dump` 导出的数据并不是 OpenMetrics 格式。实际上，它是一种更为底层、直接反映存储结构的格式，主要用于调试和低级数据操作。

Prometheus [2.24.0 / 2021-01-06](https://github.com/prometheus/prometheus/blob/main/CHANGELOG.md#2240--2021-01-06) 支持了 [backfilling](https://prometheus.io/docs/prometheus/latest/storage/#backfilling-from-openmetrics-format) 功能（能够将 OpenMetrics 格式的数据导入到 Prometheus）。

于是有人 [提议](https://github.com/prometheus/prometheus/issues/8280) `promtool tsdb dump` 导出的数据如果是 OpenMetrics 格式，那么可以实现更方便的导入和导出。Prometheus 作者的 [意见](https://github.com/prometheus/prometheus/issues/8280#issuecomment-743888219)，迁移数据的正确方式就是 tsdb 的快照功能。

## 总结

如果使用 `promtool tsdb dump` 导出，然后通过 shell/Python 转换成 OpenMetrics 标准格式文本文件，就能够通过我们的 `promtool` 导入，测试已经导入成功。问题是 `promtool tsdb dump` 此命令不能对数据进行筛选，会导致无意义的耗时。

## 方案四：使用 API 查询指定时间段指标，转换成 OpenMetrics

Prometheus [2.24.0 / 2021-01-06](https://github.com/prometheus/prometheus/blob/main/CHANGELOG.md#2240--2021-01-06) 支持了 [backfilling](https://prometheus.io/docs/prometheus/latest/storage/#backfilling-from-openmetrics-format) 功能（能够将 OpenMetrics 格式的数据导入到 Prometheus）。

通过 API 查询到想要的数据（能够自定义时间范围、自定义指标名称、自定义步长），没有多余的耗时。

## 总结

以上代码顺利查询指标并导出，仅依赖 `requests`。如果有必要，或许能够通过 shell 处理，能够进一步减少依赖，直接运行即可。导出 200 多条时间序列，前 30 小时的数据，最后文本文件的大小不到 500M，在可以接受的范围。主要原因是 OpenMetrics 格式中存在太多重复字段。

```python
# -*- coding: utf-8 -*-
# !/usr/bin/python
import multiprocessing
import subprocess
import threading
import datetime
import tarfile
import shutil
import json
import gzip
import time
import sys
import re
import os
import io
from multiprocessing.pool import ThreadPool
from multiprocessing import Pool
from threading import Thread

if sys.version_info.major == 2:
    from Queue import Queue, Empty
    from urllib import quote

    reload(sys)
    sys.setdefaultencoding('utf-8')
else:
    from queue import Queue, Empty
    from urllib.parse import quote

GRAFANA_METRICS_BASE = [
    "elasticsearch_jvm_gc_collection_seconds_count",
    "elasticsearch_jvm_gc_collection_seconds_sum",
    "elasticsearch_jvm_memory_max_bytes",
    "elasticsearch_jvm_memory_pool_peak_used_bytes",
    "elasticsearch_jvm_memory_pool_used_bytes",
    "elasticsearch_jvm_memory_used_bytes",
    "elasticsearch_nodes_roles",
    "elasticsearch_os_cpu_percent",
    "elasticsearch_os_load1",
    "elasticsearch_os_load15",
    "elasticsearch_os_load5",
    "elasticsearch_process_open_files_count",
    "elasticsearch_search_active_queries",
    "elasticsearch_thread_pool_active_count",
    "elasticsearch_thread_pool_completed_count",
    "elasticsearch_thread_pool_queue_count",
    "elasticsearch_thread_pool_rejected_count",
    "elasticsearch_transport_rx_size_bytes_total",
    "elasticsearch_transport_tx_size_bytes_total",
    "elasticsearch_breakers_estimated_size_bytes",
    "elasticsearch_breakers_limit_size_bytes",
    "elasticsearch_breakers_tripped"
]
GRAFANA_METRICS_FILE = [
    "elasticsearch_filesystem_data_available_bytes",
    "elasticsearch_filesystem_data_size_bytes",
    "elasticsearch_filesystem_data_used_percent",
    "elasticsearch_filesystem_io_stats_device_operations_count",
    "elasticsearch_filesystem_io_stats_device_read_operations_count",
    "elasticsearch_filesystem_io_stats_device_read_size_kilobytes_sum",
    "elasticsearch_filesystem_io_stats_device_write_operations_count",
    "elasticsearch_filesystem_io_stats_device_write_size_kilobytes_sum"
]
GRAFANA_METRICS_CLUSTER = [
    "elasticsearch_cluster_health_active_primary_shards",
    "elasticsearch_cluster_health_active_shards",
    "elasticsearch_cluster_health_delayed_unassigned_shards",
    "elasticsearch_cluster_health_initializing_shards",
    "elasticsearch_cluster_health_number_of_data_nodes",
    "elasticsearch_cluster_health_number_of_nodes",
    "elasticsearch_cluster_health_number_of_pending_tasks",
    "elasticsearch_cluster_health_relocating_shards",
    "elasticsearch_cluster_health_status",
    "elasticsearch_cluster_health_unassigned_shards"
]
GRAFANA_METRICS_XDCR = [
    "elasticsearch_xdcr_tasks_index_number",
    "elasticsearch_xdcr_tasks_repository",
    "elasticsearch_cat_xdcr_localDocs",
    "elasticsearch_cat_xdcr_localGCP",
    "elasticsearch_cat_xdcr_localLCP",
    "elasticsearch_cat_xdcr_localSeqNo",
    "elasticsearch_cat_xdcr_remoteDocs",
    "elasticsearch_cat_xdcr_remoteGCP",
    "elasticsearch_cat_xdcr_remoteLCP",
    "elasticsearch_cat_xdcr_remoteSeqNo"
]
GRAFANA_METRICS_ABOUT_NODE_COUNT = [
    "elasticsearch_indices_docs",
    "elasticsearch_indices_docs_deleted",
    "elasticsearch_indices_fielddata_evictions",
    "elasticsearch_indices_fielddata_memory_size_bytes",
    "elasticsearch_indices_filter_cache_evictions",
    "elasticsearch_indices_flush_time_seconds",
    "elasticsearch_indices_flush_total",
    "elasticsearch_indices_get_exists_time_seconds",
    "elasticsearch_indices_get_exists_total",
    "elasticsearch_indices_get_missing_time_seconds",
    "elasticsearch_indices_get_missing_total",
    "elasticsearch_indices_get_time_seconds",
    "elasticsearch_indices_get_total",
    "elasticsearch_indices_indexing_delete_time_seconds_total",
    "elasticsearch_indices_indexing_delete_total",
    "elasticsearch_indices_indexing_index_time_seconds_total",
    "elasticsearch_indices_indexing_index_total",
    "elasticsearch_indices_merges_docs_total",
    "elasticsearch_indices_merges_total",
    "elasticsearch_indices_merges_total_size_bytes_total",
    "elasticsearch_indices_merges_total_time_seconds_total",
    "elasticsearch_indices_query_cache_evictions",
    "elasticsearch_indices_query_cache_memory_size_bytes",
    "elasticsearch_indices_refresh_time_seconds_total",
    "elasticsearch_indices_refresh_total",
    "elasticsearch_indices_search_fetch_time_seconds",
    "elasticsearch_indices_search_fetch_total",
    "elasticsearch_indices_search_query_time_seconds",
    "elasticsearch_indices_search_query_total",
    "elasticsearch_indices_segments_count",
    "elasticsearch_indices_segments_memory_bytes",
    "elasticsearch_indices_store_size_bytes",
    "elasticsearch_indices_store_throttle_time_seconds_total",
    "elasticsearch_indices_translog_operations",
    "elasticsearch_indices_translog_size_in_bytes"
]
GRAFANA_METRICS_ABOUT_INDEX_COUNT = [
    "elasticsearch_index_stats_index_current",
    "elasticsearch_index_stats_indexing_index_time_seconds_total",
    "elasticsearch_index_stats_indexing_index_total",
    "elasticsearch_index_stats_search_fetch_time_seconds_total",
    "elasticsearch_index_stats_search_query_time_seconds_total",
    "elasticsearch_index_stats_search_query_total",
    "elasticsearch_indices_deleted_docs_total",
    "elasticsearch_indices_docs_primary",
    "elasticsearch_indices_docs_total",
    "elasticsearch_indices_segment_count_primary",
    "elasticsearch_indices_segment_count_total",
    "elasticsearch_indices_segment_doc_values_memory_bytes_primary",
    "elasticsearch_indices_segment_doc_values_memory_bytes_total",
    "elasticsearch_indices_segment_fields_memory_bytes_primary",
    "elasticsearch_indices_segment_fields_memory_bytes_total",
    "elasticsearch_indices_segment_fixed_bit_set_memory_bytes_primary",
    "elasticsearch_indices_segment_fixed_bit_set_memory_bytes_total",
    "elasticsearch_indices_segment_index_writer_memory_bytes_primary",
    "elasticsearch_indices_segment_index_writer_memory_bytes_total",
    "elasticsearch_indices_segment_memory_bytes_primary",
    "elasticsearch_indices_segment_memory_bytes_total",
    "elasticsearch_indices_segment_norms_memory_bytes_primary",
    "elasticsearch_indices_segment_norms_memory_bytes_total",
    "elasticsearch_indices_segment_points_memory_bytes_primary",
    "elasticsearch_indices_segment_points_memory_bytes_total",
    "elasticsearch_indices_segment_terms_memory_primary",
    "elasticsearch_indices_segment_terms_memory_total",
    "elasticsearch_indices_segment_version_map_memory_bytes_primary",
    "elasticsearch_indices_segment_version_map_memory_bytes_total",
    "elasticsearch_indices_store_size_bytes_primary",
    "elasticsearch_indices_store_size_bytes_total"
]
GRAFANA_METRICS = {
    '1': GRAFANA_METRICS_BASE,
    '2': GRAFANA_METRICS_FILE,
    '3': GRAFANA_METRICS_CLUSTER,
    '4': GRAFANA_METRICS_XDCR,
    '5': GRAFANA_METRICS_ABOUT_NODE_COUNT,
    '6': GRAFANA_METRICS_ABOUT_INDEX_COUNT
}
GLOBAL_CPU_CORES = None
GLOBAL_THREAD_COUNT = None

log_lock = threading.Lock()

session = None
"""设置 session，降低请求耗时"""

def setup_session():
    global session
    try:
        import requests
        from requests.adapters import HTTPAdapter
        from urllib3.util.retry import Retry
        from requests.packages.urllib3.exceptions import InsecureRequestWarning
        session = requests.Session()
        retries = Retry(total=5, backoff_factor=0.1, status_forcelist=[500, 502, 503, 504])
        session.mount('https://', HTTPAdapter(max_retries=retries))
        requests.packages.urllib3.disable_warnings(InsecureRequestWarning)
    except ImportError:
        session = None

def setting_threading():
    global GLOBAL_THREAD_COUNT
    global GLOBAL_CPU_CORES
    GLOBAL_CPU_CORES = multiprocessing.cpu_count()
    log_info("当前机器的 CPU 核心数是 {}。".format(GLOBAL_CPU_CORES))
    log_info("设置合理的线程数，取决于本机的核心数，也取决于 Prometheus 的负载压力。")
    while True:
        threading_input = one_input("请输入希望设置的线程数(建议值为核心数的 2 倍，回车确认): ")
        if not threading_input:
            GLOBAL_THREAD_COUNT = GLOBAL_CPU_CORES * 2
            break
        try:
            threading_count = int(threading_input)
            if threading_count > 0:
                GLOBAL_THREAD_COUNT = threading_count
                break
            else:
                log_warning("线程数必须是正整数，请重新输入。")
        except ValueError:
            log_warning("无效输入，请输入一个正整数。")
    return GLOBAL_THREAD_COUNT

def one_input(prompt):
    return raw_input(prompt) if sys.version_info.major == 2 else input(prompt)

def log_info(message, *args):
    """为了防止多线程打日志太快造成混乱"""
    with log_lock:
        print('[[ INFO ]] ' + message.format(*args))

def log_warning(message, *args):
    with log_lock:
        print('[[ WARNING ]] ' + message.format(*args))

def log_error(message, *args):
    with log_lock:
        print('[[ ERROR ]] ' + message.format(*args))

def query_prometheus(url, auth, headers=None):
    """通用的请求函数，支持 requests 和 curl"""
    try:
        if session:
            response = session.get(url, auth=auth, verify=False, headers=headers)
            response.raise_for_status()
            log_info("request 请求的链接是 {}", response.url)
            return response.json()
        else:
            cmd = [
                'curl', '-s', '-k', '-L',  # 静默输出，忽略证书，支持重定向
                '--keepalive',  # 允许在多个请求之间重用 TCP 连接
                '--compressed',  # 接受压缩的响应
                '--connect-timeout', '5',  # 设置连接超时为 5 秒
                '--max-time', '10',  # 设置请求最大持续时间为 10 秒
                '--retry', '3',  # 设置重试次数为 3
                '--retry-delay', '2',  # 设置重试间隔为 2 秒
            ]
            if auth is not None:
                cmd += ['-u', '{}:{}'.format(auth[0], auth[1])]
            if headers is not None:
                for key, value in headers.items():
                    cmd += ['-H', '{}: {}'.format(key, value)]
            cmd.append(url)
            log_info("curl 请求的链接是 {}", subprocess.list2cmdline(cmd))
            response = subprocess.check_output(cmd)
            return json.loads(response)
    except Exception as e:
        log_error("获取数据失败，请检查 {}，原因: {}", url, str(e))
        return None

def setting_prometheus_url():
    """等待用户输入 Prometheus_url"""
    while True:
        prometheus_url = one_input("请输入要查询的 Prometheus 地址(域名或 IP+端口，结尾不要斜线):")
        if re.match(r'^https?://', prometheus_url):
            return prometheus_url
        else:
            log_warning("请输入正确的地址，形如 https://xxx.com 或者 http://x.x.x.x:9090")

def setting_auth():
    """等待用户输入认证"""
    while True:
        username = one_input("请输入要查询的 Prometheus 用户名/无认证可回车跳过:")
        password = one_input("请输入要查询的 Prometheus 密码/无认证可回车跳过:")
        if username and not password:
            log_warning("请提供完整的用户名和密码/无认证可回车跳过。")
            continue
        elif not username and password:
            log_warning("请提供完整的用户名和密码/无认证可回车跳过。")
            continue
        auth = (username, password) if username and password else None
        return auth

def setting_time(step):
    """等待用户输入查询的时间范围或小时数"""
    while True:
        time_str = one_input("请输入往前查的时间范围，例如：20231111-20231112 或者小时数:")
        if re.match(r'\d{8}-\d{8}', time_str):
            start_date, end_date = time_str[:8], time_str[9:]
            try:
                start_time = datetime.datetime.strptime(start_date, "%Y%m%d").strftime("%Y-%m-%dT%H:%M:%SZ")
                end_time = datetime.datetime.strptime(end_date, "%Y%m%d").strftime("%Y-%m-%dT%H:%M:%SZ")
                hours = int((datetime.datetime.strptime(end_date, "%Y%m%d") -
                             datetime.datetime.strptime(start_date, "%Y%m%d")).total_seconds() / 3600)
                if check_query_limit(hours, step):
                    log_warning("查询数据量超过 11000，请重新输入")
                else:
                    return start_time, end_time, hours
            except ValueError:
                log_warning("请输入正确的日期格式，例如：20231111-20231112")
        elif time_str.isdigit():
            hours = int(time_str)
            if check_query_limit(hours, step):
                log_warning("查询数据量超过 11000，请重新输入")
            else:
                end_time = datetime.datetime.now().strftime("%Y-%m-%dT%H:%M:%SZ")
                query_time = datetime.datetime.now() - datetime.timedelta(hours=hours)
                start_time = query_time.strftime("%Y-%m-%dT%H:%M:%SZ")
                return start_time, end_time, hours
        else:
            log_warning("请输入正确的时间范围，例如：20231111-20231112 或者小时数")

def check_query_limit(hours, step):
    """检查是否超出 Prometheus 限制"""
    return hours * 60 * 60 / step > 11000

def get_clusters(prometheus_url, auth):
    """查询 Prometheus 中全部的集群列表"""
    cluster_url = '{}/api/v1/label/cluster/values'.format(prometheus_url)
    clusters = query_prometheus(cluster_url, auth)
    if not clusters:
        log_error("没有查询到任何集群")
        sys.exit(1)
    print("监控的集群有:")
    for cluster in clusters['data']:
        print(cluster)
    return clusters

def setting_cluster(clusters):
    """等待用户输入要查询的集群"""
    while True:
        cluster = one_input("请输入要查询集群名称:")
        if clusters and cluster in clusters['data']:
            return cluster
        else:
            log_warning("输入不存在，请输入正确的集群名称")

def get_scrape_interval(config, cluster):
    """匹配配置文件中的全局 scrape_interval 和每个集群的 scrape_interval"""
    scrape_interval_pattern = re.compile(
        r'job_name:\s+.*?{}/.*?\s+.*?scrape_interval:\s+(\d+)s'.format(re.escape(cluster)),
        re.DOTALL
    )
    scrape_interval_match = scrape_interval_pattern.search(config)
    if scrape_interval_match:
        return int(scrape_interval_match.group(1))
    global_scrape_interval_pattern = re.compile(r'global:\s+scrape_interval:\s+(\d+)s')
    global_scrape_interval_match = global_scrape_interval_pattern.search(config)
    if global_scrape_interval_match:
        return int(global_scrape_interval_match.group(1))
    return None

def setting_step(prometheus_url, auth, cluster):
    """等待用户设置查询间隔"""
    config_url = '{}/api/v1/status/config'.format(prometheus_url)
    config = query_prometheus(config_url, auth)['data']['yaml']
    scrape_interval = get_scrape_interval(config, cluster)
    if scrape_interval is None:
        log_warning("无法找到集群 {} 或全局配置的爬取间隔".format(cluster))
        return None
    while True:
        step_input = one_input(
            "集群 {} 配置的爬取间隔是 {} 秒，回车使用默认值，或输入任意正整数值:".format(cluster, scrape_interval)
        )
        if not step_input:
            log_info("最大可以往前查 {} 小时".format(int(11000 / (3600 / scrape_interval))))
            return scrape_interval
        try:
            step = int(step_input)
            if step > 0:
                log_info("最大可以往前查 {} 小时".format(int(11000 / (3600 / step))))
                return step
            else:
                log_warning("请输入一个正整数")
        except ValueError:
            log_warning("请输入一个正整数")

def setting_metrics(step, hours):
    """等待用户选择要查询的指标"""
    total_query_count = int(3600 / step * hours)
    metrics_message = ("""输入数字代表要查询的指标集(无需分隔)
1. jvm_gc_thread_pool (jvm/gc 相关指标，建议选择)
2. filesystem_io (文件系统相关指标，建议选择)
3. cluster_health (集群健康相关指标，建议选择)
4. xdcr_tasks (未开启跨集群复制，请勿选择此项)
5. elasticsearch_indices.* (每个 elasticsearch_indices 开头的指标，有 [[{}*节点数]] 行数据)
6. elasticsearch_index.* (每个 elasticsearch_indices 开头的指标，有 [[{}*索引数]] 行数据)
[[ WARNING ]] 在下面的步骤中，可以自定义输入要额外查询的指标，但是请记住上面的两条"""
                       .format(total_query_count, total_query_count))
    print(metrics_message)
    metrics_list = []
    unique_options = []
    while True:
        options = one_input("请输入要查询集合如 123，回车跳过: ")
        if not options:
            log_warning("已跳过，请在下一轮输入准确的所需指标名")
            break
        if not options.isdigit():
            log_warning("输入包含不合规字符，请重新输入。")
            continue
        invalid_metrics = [metric for metric in options if metric not in GRAFANA_METRICS]
        if invalid_metrics:
            log_warning("不合规的输入: {}".format(' '.join(invalid_metrics)))
            continue
        for single_option in options:
            if single_option not in unique_options:
                unique_options.append(single_option)
                metrics_list.extend(GRAFANA_METRICS[single_option])
        break
    return metrics_list, unique_options

def get_all_metrics(prometheus_url, auth):
    """查询 Prometheus 中有哪些 metrics"""
    series_url = '{}/api/v1/label/__name__/values'.format(prometheus_url)
    try:
        series = query_prometheus(series_url, auth)
        if not series:
            raise ValueError("获取指标失败，返回结果为空")
        all_metrics = [metric for metric in series.get('data', []) if 'elasticsearch' in metric]
        return all_metrics
    except Exception as e:
        log_error("查询指标时发生错误: {}".format(e))
        sys.exit(1)

def setting_extra_metrics(metrics_list, prometheus_url, auth):
    """列出所有可用且未被选择的 Elasticsearch 指标供用户选择，支持用户添加额外的查询指标"""
    all_metrics = get_all_metrics(prometheus_url, auth)
    available_metrics = set(all_metrics) - set(metrics_list)
    if available_metrics:
        log_info("以下是未添加的可用的 Elasticsearch 指标列表，可以在指导下添加：")
        print(",".join(sorted(available_metrics)))
        while True:
            add_metrics = one_input("可以在指导下补充额外的查询指标，逗号分隔，回车跳过:")
            if not add_metrics:
                break
            add_metrics_list = [metric.strip() for metric in add_metrics.split(',')]
            for metric in add_metrics_list:
                if metric:
                    if metric in metrics_list:
                        log_info("{} 指标已经添加，跳过".format(metric))
                    elif metric in all_metrics:
                        metrics_list.append(metric)
                        log_info("添加指标：{}".format(metric))
                    else:
                        log_warning("不支持的指标 {}".format(metric))
    else:
        log_info("所有的 Elasticsearch 指标已添加。")
    return metrics_list

def get_active_indices(prometheus_url, auth, cluster, start_time, hours):
    top_n = None
    log_info("为了提高查询效率，我们提供了查看特定数量最活跃或最大的索引的功能。")
    while top_n is None:
        user_input = one_input("请输入要查看的索引数量,(仅列出,供复制)")
        if user_input.isdigit():
            top_n = int(user_input)
        else:
            log_warning("输入错误，请输入一个有效的数字表示要查看的索引数量。")
    start_time_dt = datetime.datetime.strptime(start_time, '%Y-%m-%dT%H:%M:%SZ')
    end_time_dt = start_time_dt + datetime.timedelta(hours=hours)
    end_time_str = end_time_dt.strftime('%Y-%m-%dT%H:%M:%SZ')
    top_docs_query = (
        'topk({top_n}, max_over_time(elasticsearch_indices_docs_total{{cluster="{cluster}"}}[{hours}h]))'
        .format(top_n=top_n, cluster=cluster, hours=hours))
    top_docs_query = quote(top_docs_query)
    top_docs_url = ('{}/api/v1/query?query={}&time={}'
                    .format(prometheus_url, top_docs_query, end_time_str))
    top_docs_data = query_prometheus(top_docs_url, auth)
    if not top_docs_data:
        log_error("查询文档数最多的索引失败，请检查了是否采集了 elasticsearch_indices_docs_total 指标")
        sys.exit(1)
    top_activity_query = (
        'topk({top_n}, max_over_time(elasticsearch_index_stats_indexing_index_total{{cluster="{cluster}"}}[{hours}h]) '
        '+ max_over_time(elasticsearch_index_stats_search_query_total{{cluster="{cluster}"}}[{hours}h]))'
        .format(top_n=top_n, cluster=cluster, hours=hours))
    top_activity_query = quote(top_activity_query)
    top_activity_url = ('{}/api/v1/query?query={}&time={}'
                        .format(prometheus_url, top_activity_query, end_time_str))
    top_activity_data = query_prometheus(top_activity_url, auth)
    if not top_activity_data:
        log_error("查询活跃度最高的索引失败，请检查是否采集了 elasticsearch_index_stats_indexing_index_total 指标")
        sys.exit(1)
    top_docs_indices = [result['metric']['index'] for result in top_docs_data['data']['result']]
    top_activity_indices = [result['metric']['index'] for result in top_activity_data['data']['result']]
    big_and_active_indices = set(top_docs_indices) | set(top_activity_indices)
    log_info("文档数最多的前 {} 个索引:\n{}".format(len(top_docs_indices), ', '.join(top_docs_indices)))
    log_info("活跃度最高的前 {} 个索引:\n{}".format(len(top_activity_indices), ', '.join(top_activity_indices)))
    log_info("文档数最多和活跃度最高的前 {} 个索引:\n{}"
             .format(len(big_and_active_indices), ', '.join(big_and_active_indices)))
    return top_docs_indices, top_activity_indices, big_and_active_indices

def setting_indices(prometheus_url, auth, cluster, start_time, hours, unique_options):
    """等待用户选择要查询的索引"""
    indices_list = []
    if '6' in unique_options:
        top_docs_indices, top_activity_indices, big_and_active_indices = get_active_indices(
            prometheus_url, auth, cluster, start_time, hours)
        while True:
            indices = one_input("选择了查询特定索引的指标(直接回车可以取消)，请指定要查询的索引，逗号分隔，回车结束:")
            if not indices:
                log_info("没有输入任何索引，将取消选项 6 的查询。")
                unique_options.remove('6')
                break
            else:
                indices_list = [index.strip() for index in indices.split(',') if index.strip()]
                if all(index in big_and_active_indices for index in indices_list):
                    log_info("已选择索引: {}".format(", ".join(indices_list)))
                    return indices_list, unique_options
                else:
                    indices_list = []
                    log_warning("输入的索引不存在于最活跃或最大的索引列表中，请重新输入有效的索引。")
    return indices_list, unique_options

def generate_promql(cluster, metrics_list, indices_list, metrics_with_index):
    """promql 生成，如果是 pass 就使用 pod，如果是不 pass 就使用 cluster，如果指标带有 index 并且提供了 indices 就填充 promql"""
    promql_list = []
    template_with_index = '{}{{cluster=~"{}.*", index="{}"}}'
    template_without_index = '{}{{cluster=~"{}.*"}}'
    for metric in metrics_list:
        needs_index = metric in GRAFANA_METRICS['6'] or metric in metrics_with_index
        if needs_index and indices_list:
            for index in indices_list:
                promql_list.append(template_with_index.format(metric, cluster, index))
        else:
            promql_list.append(template_without_index.format(metric, cluster))
    if not promql_list:
        log_error("没有生成任何 PromQL 查询")
        sys.exit(1)
    return promql_list

def get_buffer_size():
    """计算合适的文件缓冲区大小。"""
    mem_bytes = os.sysconf('SC_PAGE_SIZE') * os.sysconf('SC_PHYS_PAGES')
    mem_gib = mem_bytes / (1024 ** 3)
    if mem_gib >= 16:
        return 262144
    elif mem_gib >= 8:
        return 131072
    elif mem_gib >= 4:
        return 65536
    else:
        return 32768

def get_metrics_with_index_label(prometheus_url, auth, start_time):
    """查询 Prometheus 中在指定时间范围内所有带有 index 标签的指标"""
    series_api_url = ("{}/api/v1/series?match[]=%7Bindex%3D~%22.%2B%22%7D&start={}&end={}"
                      .format(prometheus_url, quote(start_time), quote(start_time)))
    try:
        data = query_prometheus(series_api_url, auth)
        metrics_with_index = set()
        for series in data.get('data', []):
            metric_name = series['__name__']
            metrics_with_index.add(metric_name)
        return metrics_with_index
    except Exception as e:
        log_error("查询带有 index 的指标时发生错误: {}".format(e))
        sys.exit(1)

def get_metrics(prometheus_url, auth, promql_list, start_time, end_time, cluster, unique_options, hours, step):
    """使用线程池对 Prometheus 进行请求，计算进度，使用队列边查边写边压缩，节省时间"""
    metrics_data_queue = Queue()
    query_pool = ThreadPool(GLOBAL_THREAD_COUNT)
    write_thread = Thread(target=write_metrics_to_file,
                          args=(metrics_data_queue, cluster, unique_options, hours, step, promql_list))
    write_thread.start()
    tasks = []
    for promql in promql_list:
        encoded_promql = quote(promql)
        metrics_url = "{}/api/v1/query_range?query={}&start={}&end={}&step={}s".format(
            prometheus_url, encoded_promql, start_time, end_time, step)
        headers = {'Content-Type': 'application/json'}
        task = query_pool.apply_async(query_prometheus, (metrics_url, auth, headers))
        tasks.append(task)
    total_queries = len(tasks)
    queries_done = 0
    try:
        for task in tasks:
            try:
                prometheus_data = task.get(timeout=300)
                if 'data' in prometheus_data and 'result' in prometheus_data['data']:
                    for result in prometheus_data['data']['result']:
                        metrics_data_queue.put(result)
                queries_done += 1
                if queries_done % 10 == 0 or queries_done == total_queries:
                    percent_complete = (queries_done / total_queries) * 100
                    log_info(
                        '#################  进度: {}/{} ({:.2f}%)'.format(queries_done, total_queries, percent_complete))
            except Exception as e:
                log_error("解析数据出错 {}".format(e))
                queries_done += 1
                if queries_done % 10 == 0 or queries_done == total_queries:
                    percent_complete = (queries_done / total_queries) * 100
                    log_info(
                        '#################  进度: {}/{} ({:.2f}%)'.format(queries_done, total_queries, percent_complete))
    except Exception as e:
        log_error('在处理任务时发生错误: {}'.format(e))
    finally:
        metrics_data_queue.put(None)
        query_pool.close()
        query_pool.join()
    write_thread.join()

def write_metrics_to_file(metrics_data_queue, cluster, unique_options, hours, step, promql_list):
    """将指标写入文件，按文件行数分割后压缩"""
    compress_pool = Pool(GLOBAL_CPU_CORES // 2)
    unique_options = ''.join(unique_options)
    file_index = 1
    line_count = 0
    total_query_count = int(3600 / step * hours)
    total_line = total_query_count * len(promql_list)
    file_count = 32
    max_lines_per_file = total_line // file_count
    max_lines_per_file = max(max_lines_per_file, 50000)
    filename_pattern = "openmetrics{}_{}_{}_{}h_{}.txt"
    filename = filename_pattern.format(file_index, cluster, unique_options, hours, step)
    buffer_size = get_buffer_size()
    with io.open(filename, 'w', buffering=buffer_size) as f:
        while True:
            try:
                result = metrics_data_queue.get(timeout=10)
            except Empty:
                log_warning("队列已经 10 秒没有数据了，检查是否所有查询都已经完成。")
                continue
            if result is None:
                break
            metric_name = result['metric']['__name__']
            labels = ','.join(
                '{}="{}"'.format(key, value) for key, value in result['metric'].items() if key != '__name__')
            for value in result['values']:
                timestamp, metric_value = value
                if line_count >= max_lines_per_file:
                    f.write('# EOF\n')
                    f.flush()
                    f.close()
                    compress_pool.apply_async(compress_single, (filename,))
                    file_index += 1
                    filename = filename_pattern.format(file_index, cluster, unique_options, hours, step)
                    f = io.open(filename, 'w', buffering=buffer_size)
                    line_count = 0
                f.write(u"{}{{{}}} {} {}\n".format(metric_name, labels, metric_value, timestamp))
                line_count += 1
    f.write('# EOF\n')
    f.flush()
    f.close()
    compress_pool.apply_async(compress_single, (filename,))
    compress_pool.close()
    compress_pool.join()

def compress_single(filename):
    """压缩单个 txt 文件，压缩后删除"""
    print("开始压缩文件: {}, 请不要终止脚本".format(filename))
    file_size = os.path.getsize(filename)
    if file_size > 1024 * 1024:
        gzip_filename = filename + '.gz'
        with open(filename, 'rb') as f_in, gzip.open(gzip_filename, 'wb') as f_out:
            shutil.copyfileobj(f_in, f_out)
        os.remove(filename)

def compress_all(directory, output_filename, cluster, unique_options, hours, step):
    """打包压缩全部符合格式的 gz 文件"""
    unique_options = ''.join(unique_options)
    expected_filename_part = "{}_{}_{}h_{}".format(cluster, unique_options, hours, step)
    with tarfile.open(output_filename, "w:gz") as tar:
        for filename in os.listdir(directory):
            if filename.endswith('.gz') and expected_filename_part in filename:
                tar.add(os.path.join(directory, filename), arcname=filename)

def initialize(step, prometheus_url, auth):
    """初始化，拿到用户输入的时间和选择的指标列表和选择项"""
    start_time, end_time, hours = setting_time(step)
    metrics_list, unique_options = setting_metrics(step, hours)
    metrics_list = setting_extra_metrics(metrics_list, prometheus_url, auth)
    return start_time, end_time, hours, metrics_list, unique_options

def main():
    intro_message = """##############################################################
脚本将通过 Prometheus 内置的 API 接口导出 Elasticsearch 相关的监控数据
只会获取 Elasticsearch 的各项状态信息，不会获取 Elasticsearch 的数据
设置的爬取间隔越大，能够查询的范围就越长，需要保证所在路径空间大于 30G
脚本会对 Prometheus 造成小幅查询压力，运行前请检查 Prometheus 负载状态
脚本涉及到压缩和批量请求，在配置更高的机器上，导出的效率更高，耗时更短
脚本导出后，开始压缩不要终止，将当前路径的 transportMe.tar.gz 传回即可
##############################################################"""
    print(intro_message)
    setup_session()
    prometheus_url = setting_prometheus_url()
    auth = setting_auth()
    setting_threading()
    clusters = get_clusters(prometheus_url, auth)
    cluster = setting_cluster(clusters)
    step = setting_step(prometheus_url, auth, cluster)
    start_time, end_time, hours, metrics_list, unique_options = initialize(step, prometheus_url, auth)
    indices_list, unique_options = setting_indices(prometheus_url, auth, cluster, start_time, hours, unique_options)
    metrics_with_index = get_metrics_with_index_label(prometheus_url, auth, start_time)
    promql_list = generate_promql(cluster, metrics_list, indices_list, metrics_with_index)
    start_query_time = time.time()
    get_metrics(prometheus_url, auth, promql_list, start_time, end_time, cluster, unique_options, hours, step)
    compress_all(".", "transportMe.tar.gz", cluster, unique_options, hours, step)
    end_query_time = time.time()
    query_duration = round(end_query_time - start_query_time, 2)
    print("导出成功，总耗时 {} 秒，将 transportMe.tar.gz 传回即可".format(query_duration))

if __name__ == "__main__":
    main()
```

目前可行性最高的方案，本地测试和测试环境测试均已成功导出导入。由于 OpenMetrics 重复率较高，有较高的压缩比例，4GB 的文本文件可以压缩为 40MB，导入到本地的 Prometheus，重启 Prometheus（让其重载块文件）在 Prometheus 和 Grafana 均可正常查询。

## 耗时估算

- 测试环境：10 个节点，846 个索引，使用 `requests`，耗时约 4 分钟
- 真实环境：61 个节点，2168 个索引，使用 `requests`，耗时约 17 分钟

## 更新记录

- **2023.12.4 更新**：新增了一种请求方式，即使没有 [requests](https://requests.readthedocs.io/en/latest/)，也可以通过 `curl` 来发送请求，没有任何的外部依赖
- **2023.12.8 更新**：新增了错误重输的功能，当用户输入不满足要求时，会提示重新输入并不会脚本退出；新增了指定时间范围查询，假如故障发生在上周，既可以接受小时数输入也可以接受日期范围的输入
- **2023.12.14 更新**：新增了支持额外添加指标的功能，目前 Grafana 看板并没有用到 exporter 采集的全部指标；新增了分段查询的功能，最后写入多个文件，能够在导入时提高导入速度，并且能够实现同步的查询压缩写入
- **2023.12.19 更新**：新增了索引过滤功能，利用指标列举出被监控的索引中最大的、最活跃的，供用户选择复制；新增了线程池，加快了请求 Prometheus 的速度，会小幅增加 Prometheus 的查询压力；新增了查询进度的估算功能，用户可根据进度估算耗时，决定是否取消查询