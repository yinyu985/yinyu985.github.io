---
title: Grafana 日志聚合工具 Loki 搭建
slug: building-grafana-log-aggregation-tool-loki
tags: [Grafana, Loki]
date: 2022-09-13T21:58:21+08:00
---

Loki 是 Grafana Labs 团队最新的开源项目，是一个水平可扩展、高可用性、多租户的日志聚合系统。它的设计非常经济高效且易于操作，因为它不会为日志内容编制索引，而是为每个日志流编制一组标签，为 Prometheus 和 Kubernetes 用户做了相关优化。项目受 Prometheus 启发，类似于 Prometheus 的日志系统。<!--more-->

Loki 的架构非常简单，使用了和 Prometheus 一样的标签来作为索引，也就是说，你通过这些标签既可以查询日志的内容也可以查询到监控的数据，不但减少了两种查询之间的切换成本，也极大地降低了日志索引的存储。Loki 将使用与 Prometheus 相同的服务发现和标签重新标记库，编写了 Promtail，在 k8s 中 Promtail 以 DaemonSet 方式运行在每个节点中，通过 Kubernetes API 获取日志的正确元数据，并将它们发送到 Loki。

## 特点

与其他日志聚合系统相比，Loki 具有下面的一些特性：

- 不对日志进行全文索引。通过存储压缩非结构化日志和仅索引元数据，Loki 操作起来会更简单，更省成本。
- 通过使用与 Prometheus 相同的标签记录流对日志进行索引和分组，这使得日志的扩展和操作效率更高。
- 特别适合储存 Kubernetes Pod 日志；诸如 Pod 标签之类的元数据会被自动采集和编入索引。
- 受 Grafana 原生支持。

## 组成

- `loki` 是主服务器，负责存储日志和处理查询。
- `promtail` 是代理，负责收集日志并将其发送给 `loki`。
- Grafana 用于 UI 展示。

## 对比

首先，ELK/EFK 架构功能确实强大，也经过了多年的实际环境验证，其中存储在 Elasticsearch 中的日志通常以非结构化 JSON 对象的形式存储在磁盘上，并且 Elasticsearch 为每个对象都建立了索引，以便进行全文搜索，然后用户可以使用特定查询语言来搜索这些日志数据。与之对应的 Loki 的数据存储是解耦的，既可以在磁盘上存储数据，也可以使用如 Amazon S3 的云存储系统。Loki 中的日志带有一组标签名和值，其中只有标签对被索引，这种权衡使得它比完整索引的操作成本更低，但是针对基于内容的查询，需要通过 LogQL 再单独查询。

和 Fluentd 相比，Promtail 是专门为 Loki 量身定制的，它可以为运行在同一节点上的 Kubernetes Pods 做服务发现，从指定文件夹读取日志。Loki 采用了类似于 Prometheus 的标签方式。因此，当与 Prometheus 部署在同一个环境中时，因为相同的服务发现机制，来自 Promtail 的日志通常具有与应用程序指标相同的标签，统一了标签管理。

Kibana 提供了许多可视化工具来进行数据分析，高级功能比如异常检测等机器学习功能。Grafana 专门针对 Prometheus 和 Loki 等时间序列数据打造，可以在同一个仪表板上查看日志和指标。

## 环境搭建

```bash
wget https://github.com/grafana/loki/releases/download/v2.5.0/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
mv loki-linux-amd64 /data/loki/
```

### 写配置文件

```bash
cd /data/loki/
vi loki-local-config.yaml
```

```yaml
auth_enabled: false
server:
  http_listen_port: 3100
ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 5m
  chunk_retain_period: 30s
schema_config:
  configs:
    - from: 2020-05-15
      store: boltdb
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 168h
storage_config:
  boltdb:
    directory: /data/loki/index
  filesystem:
    directory: /data/loki/chunks
limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
```

### 配置系统服务

```bash
vim /etc/systemd/system/loki.service
```

```ini
[Unit]
Description=loki
Wants=network-online.target
After=network-online.target

[Service]
Restart=on-failure
ExecStart=/data/loki/loki-linux-amd64 --config.file=/data/loki/loki-local-config.yaml

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl start loki.service
systemctl status loki.service
systemctl stop loki.service
systemctl restart loki.service
```

### 安装采集组件（安装在被采集主机）

```bash
wget https://github.com/grafana/loki/releases/download/v2.5.0/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
mv promtail-linux-amd64 /data/promtail/
```

```bash
cd /data/promtail/
vim promtail-local-config.yaml
```

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /data/promtail/positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*
```

```bash
systemctl daemon-reload
systemctl start promtail.service
systemctl status promtail.service
systemctl restart promtail.service
systemctl stop promtail.service
```

### Grafana 添加数据源

即可在 Grafana 界面通过 LogQL 语法或者鼠标点选条件，去筛选日志。

![](https://s2.loli.net/2022/08/11/E4aAWkNCeunwtXy.png)