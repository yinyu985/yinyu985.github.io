---
title: 用 Docker Compose 部署一套 Prometheus 监控系统
slug: deploy-a-prometheus-monitoring-system-using-dockercompose
tags: [Prometheus, Docker]
date: 2021-12-20T22:01:15+08:00
---

Prometheus 受启发于 Google 的 Borgmon 监控系统（相似的 Kubernetes 是从 Google 的 Borg 系统演变而来），从 2012 年开始由前 Google 工程师在 SoundCloud 以开源软件的形式进行研发，并且于 2015 年早期对外发布早期版本。2016 年 5 月继 Kubernetes 之后成为第二个正式加入 CNCF 基金会的项目，同年 6 月正式发布 1.0 版本。2017 年底发布了基于全新存储层的 2.0 版本，能更好地与容器平台、云平台配合。<!--more-->

```yaml
version: '2.2'
services:
  prometheus:
    image: prom/prometheus:v2.29.2
    container_name: prometheus
    restart: always
    privileged: true
    user: root
    ports:
      - 9090:9090
    volumes:
      - /home/docker/prometheus/:/etc/prometheus/
      - /home/docker/prometheus/conf.d:/etc/prometheus/conf.d
      - /home/docker/prometheus/data:/etc/prometheus/data
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/etc/prometheus/data'
      - '--storage.tsdb.retention=90d'
      - '--web.enable-lifecycle'

  grafana:
    privileged: true
    user: root
    image: grafana/grafana:8.1.2
    container_name: grafana
    restart: always
    ports:
      - '3000:3000'
    volumes:
      - /home/docker/grafana/data:/var/lib/grafana/
      - /home/docker/grafana/log:/var/log/grafana/
      - /home/docker/grafana/conf:/usr/share/grafana/conf

  blackbox-exporter:
    image: prom/blackbox-exporter:v0.19.0
    container_name: blackbox-exporter
    hostname: blackbox-exporter
    ports:
      - "9115:9115"
    restart: always
    volumes:
      - "/home/docker/blackbox/blackbox.yml:/config/blackbox.yml"
    command:
      - '--config.file=/config/blackbox.yml'

  node-exporter:
    image: prom/node-exporter:v1.2.2
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: always
    volumes:
      - '/:/host:ro,rslave'
    command:
      - '--path.rootfs=/host'

  openspeedtest:
    image: openspeedtest/latest:speedtest
    container_name: openspeedtest
    hostname: openspeedtest
    restart: unless-stopped
    ports:
      - '3002:3000'
      - '3001:3001'
```

少年想学 Prometheus 吗？这里有几份秘籍。  
<https://yunlzheng.gitbook.io/prometheus-book/>  
<https://prometheus.fuckcloudnative.io/>  
<https://www.kancloud.cn/huyipow/prometheus>