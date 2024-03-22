---
title: 用 Dockercompose 部署一套 Prometheus 监控系统
slug: deploy-a-prometheus-monitoring-system-using-dockercompose
tags: [ Prometheus, Docker ]
date: 2021-12-20T22:01:15+08:00
---

Prometheus受启发于Google的Brogmon监控系统（相似的Kubernetes是从Google的Brog系统演变而来），从2012年开始由前Google工程师在Soundcloud以开源软件的形式进行研发，并且于2015年早期对外发布早期版本。2016年5月继Kubernetes之后成为第二个正式加入CNCF基金会的项目，同年6月正式发布1.0版本。2017年底发布了基于全新存储层的2.0版本，能更好地与容器平台、云平台配合。<!--more-->

```
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

少年想学Prometheus吗？这里有几份秘籍。
<https://yunlzheng.gitbook.io/prometheus-book/>
<https://prometheus.fuckcloudnative.io/>
<https://www.kancloud.cn/huyipow/prometheus>
