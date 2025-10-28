---
title: Docker 运行 VictoriaMetrics 测试体验
slug: docker-run-victoriametrics-test-experience
tags: [Docker, Prometheus]
date: 2023-03-16T22:38:08+08:00
---

VictoriaMetrics 是一个开源的、快速的、低成本的时间序列数据库和监测系统。它最初是由俄罗斯 IT 公司 "Badoo" 的一个工程师团队开发的，以满足他们的监测需求。然而，它后来被作为开源软件发布，供更广泛的社区使用。VictoriaMetrics 被设计用来处理大量的时间序列数据，同时保持高性能和可扩展性。它使用一个定制的存储引擎，优化时间序列数据的存储，并实现快速查询和聚合。除了时间序列数据库的功能，VictoriaMetrics 还包括一个监控系统，可以从各种来源收集和可视化指标，包括 Prometheus、Graphite 和 InfluxDB。VictoriaMetrics 与流行的查询语言兼容，包括 PromQL 和 Graphite，并可以作为独立的二进制文件、Docker 容器或 Kubernetes 操作器部署。总的来说，VictoriaMetrics 是一个强大而灵活的工具，用于管理时间序列数据和监测系统性能，使其成为许多组织的热门选择。

<!--more-->

项目开源地址：[https://github.com/VictoriaMetrics/VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics)

VictoriaMetrics 具有以下突出特点：

- 它可以用作 Prometheus 的长期存储。有关详细信息，请参阅[这些文档](https://github.com/VictoriaMetrics/VictoriaMetrics#prometheus-setup)。

- 它可以用作 Grafana 中 Prometheus 的直接替代品，因为它支持[Prometheus querying API](https://github.com/VictoriaMetrics/VictoriaMetrics#prometheus-querying-api-usage)。

- 它可以用作 Grafana 中 Graphite 的直接替代品，因为它支持[Graphite API](https://github.com/VictoriaMetrics/VictoriaMetrics#graphite-api-usage)。

- 它为[数据摄取](https://medium.com/@valyala/high-cardinality-tsdb-benchmarks-victoriametrics-vs-timescaledb-vs-influxdb-13e6ee64dd6b)和[数据查询](https://medium.com/@valyala/when-size-matters-benchmarking-victoriametrics-vs-timescale-and-influxdb-6035811952d4)提供了高性能和良好的纵向和横向可扩展性。它[比 InfluxDB 和 TimescaleDB 高出 20 倍](https://medium.com/@valyala/measuring-vertical-scalability-for-time-series-databases-in-google-cloud-92550d78d8ae)。

- 在处理数百万个独特的时间序列（又名[高基数](https://docs.victoriametrics.com/FAQ.html#what-is-high-cardinality)）时，它[使用的 RAM 比 InfluxDB 少 10 倍](https://medium.com/@valyala/insert-benchmarks-with-inch-influxdb-vs-victoriametrics-e31a41ae2893)，[比 Prometheus、Thanos 或 Cortex 少 7 倍](https://valyala.medium.com/prometheus-vs-victoriametrics-benchmark-on-node-exporter-metrics-4ca29c75590f)。

- 它提供高数据压缩，因此根据这些基准，与 TimescaleDB 相比，可以将多达 70 倍的数据点存储到有限的存储空间中，并且根据该基准，与 Prometheus、Thanos 或 Cortex 相比，所需的存储空间最多减少 7 倍。

- 它针对具有高延迟 IO 和低 IOPS 的存储（AWS、Google Cloud、Microsoft Azure 等中的 HDD 和网络存储）进行了优化。查看[这些基准测试中的磁盘 IO 图](https://medium.com/@valyala/high-cardinality-tsdb-benchmarks-victoriametrics-vs-timescaledb-vs-influxdb-13e6ee64dd6b)。

- 单节点 VictoriaMetrics 可以替代使用 Thanos、M3DB、Cortex、InfluxDB 或 TimescaleDB 等竞争解决方案构建的中等规模集群。查看[垂直可扩展性基准](https://medium.com/@valyala/measuring-vertical-scalability-for-time-series-databases-in-google-cloud-92550d78d8ae)，[将 Thanos 与 VictoriaMetrics 集群进行比较](https://medium.com/@valyala/comparing-thanos-to-victoriametrics-cluster-b193bea1683)，以及来自[PromCon 2019 的](https://promcon.io/2019-munich/talks/remote-write-storage-wars/)[Remote Write Storage Wars](https://promcon.io/2019-munich/talks/remote-write-storage-wars/)演讲。

  ~~这个演讲我看了，来自 Adidas 的工程师，咔咔咔在 Prometheus 的技术分享会上放 Adidas 的广告，然后就开始介绍各类监控系统，各种对比，然后 Adidas 荣登 VictoriaMetrics 的用户案例首页~~

以上这些都是项目的 readme 里面自己宣传的，我只是翻译搬运，并没有做过任何测试，我能知道的就是[别人做过测试](https://segmentfault.com/a/1190000041789939)。这次使用 docker-compose 启动正是为了体验这个监控系统，为后续公司的监控系统转型做准备。话不多说，开搞！

首先前往 GitHub，这个项目的主页：[https://github.com/VictoriaMetrics/VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics)

在 master 分支下，进入到 deployment 下的 docker 文件夹，其实我们需要的都在这个 docker 文件夹里了，也就是说其实只需要下载这一个文件夹就好了。如何下载这单个文件而不是去 git clone 整个仓库呢？这里我提供一个[油猴脚本](https://greasyfork.org/zh-CN/scripts/411834-download-github-repo-sub-folder)，其实网上也是有[类似的网站](https://download-directory.github.io/)，你只需要复制文件夹路径的链接，粘贴进去，就能够提供一个下载链接，不详细展开了，自己去探索。

拿到文件夹，里面的内容已经是使用启动 VM（之后的所有 VM 都是代指 VictoriaMetrics）全部的必要条件了。我使用 VS Code 打开这个文件夹，找到 docker-compose.yaml 这个文件右键即可选择 docker-compose up，这个动作等效于，只是 VS Code 的 Docker 插件将这个动作可视化了，使用鼠标点点点，就能够启动、停止、重启，查看日志，进入容器终端，访问暴露的端口。

```bash
cd /deployment/docker
docker-compose up
```

见证奇迹，Docker 会根据当前路径的 docker-compose.yaml 文件去构建镜像，启动容器。启动成功后会有这么几个页面：

- <http://localhost:3000/> — Grafana，自带的看板
- <http://localhost:8428/> — VictoriaMetrics，提供查询和存储的主要组件
- <http://localhost:8429/> — vmagent，提供数据采集功能组件，能够直接读取 Prometheus 的配置文件
- <http://localhost:8880/> — vmalert，提供告警，去 VM 查询，符合条件就产生告警，再发送到 Alertmanager
- <http://localhost:9093/> — Alertmanager，vmalert 没有超越 Alertmanager，并只能依赖于 Alertmanager，提供告警路由、告警抑制等功能

如果你是 Prometheus 的用户，你可以前往 [https://play.victoriametrics.com/](https://play.victoriametrics.com/) 这里体验一下官方提供的一个 demo。相比于 Prometheus，这个前端页面做得确实很用心了。让我感触最深的一点就是，当你查一个指标不加任何筛选条件（即 `job="XXXX"`）可能会有很多的时间序列被查询到，它会有一个提示，因为性能原因只显示了多少条。这不是关键，你当然知道要加筛选条件，毕竟你也不想看到你不想看的序列出现在图表。当你想添加筛选条件时，你只需要点击你刚才查询的很多时间序列中的某个条件，即可复制到剪贴板，然后粘贴在上面的查询框，简直了，怎么会有这么人性化的设计。vmui 对比 Prometheus，简直完胜。

虽然 vmalert 没能超越 Alertmanager，但是其实就算做了个类 Alertmanager 的东西，不也是重复造轮子吗？毕竟 Alertmanager 已经很强大了。

我在自己的本机使用 Docker 单独运行了一个 node_exporter，然后把它添加到这套监控系统，整挺好，慢慢来吧，后续的体验记录也会继续更新在这里。

贴一篇文章：

[Thanos 与 VictoriaMetrics 集群的比较](https://faun.pub/comparing-thanos-to-victoriametrics-cluster-b193bea1683)