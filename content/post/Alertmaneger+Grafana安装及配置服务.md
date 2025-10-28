---
title: Alertmanager + Grafana 安装及配置服务
slug: alertmaneger+grafana-installation-and-configuration-service
tags: [Prometheus]
date: 2022-04-09T12:20:25+08:00
---

The [**Alertmanager**](https://github.com/prometheus/alertmanager) handles alerts sent by client applications such as the Prometheus server. It takes care of deduplicating, grouping, and routing them to the correct receiver integration such as email, PagerDuty, or OpsGenie. It also takes care of silencing and inhibition of alerts.

**Grafana** allows you to query, visualize, alert on and understand your metrics no matter where they are stored. Create, explore, and share beautiful dashboards with your team and foster a data driven culture.

<!--more-->

首先 cd /resource，移动到资源目录

```bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.23.0/alertmanager-0.23.0.linux-amd64.tar.gz
wget https://dl.grafana.com/enterprise/release/grafana-enterprise-8.2.3.linux-amd64.tar.gz
tar -xzf grafana-enterprise-8.2.3.linux-amd64.tar.gz -C ..
tar -xzf alertmanager-0.23.0.linux-amd64.tar.gz -C ..
```

为应用创建一个系统服务，通过 systemctl 来管理：

```bash
vi /usr/lib/systemd/system/alertmanager.service
```

```ini
[Unit]
Description=Alertmanager
After=network-online.target

[Service]
Restart=on-failure
ExecStart=/home/alertmanager-0.23.0.linux-amd64/alertmanager --config.file=/home/alertmanager-0.23.0.linux-amd64/alertmanager.yml

[Install]
WantedBy=multi-user.target
```

```bash
vim /usr/lib/systemd/system/grafana.service
```

```ini
[Unit]
Description=grafana
After=network.target

[Service]
Type=notify
ExecStart=/home/grafana-8.2.3/bin/grafana-server -homepath /home/grafana-8.2.3
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
systemctl start grafana    # 开启
systemctl status grafana   # 查询状态
systemctl enable grafana   # 设置开机自启
```

这个操作会将刚才创建的服务链接到 `/etc/systemd/system/` 目录下，也就是说这个目录放的都是开机自启的服务文件。

```bash
systemctl disable grafana  # 禁止开机自启，这个操作会移除上一步的那个链接，从而实现取消开机自启
systemctl daemon-reload    # 重载，添加完服务后就执行这个重载命令
systemctl reset-failed     # 重置失败的记录
```