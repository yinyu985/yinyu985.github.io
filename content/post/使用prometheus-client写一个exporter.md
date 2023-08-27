---
title: 使用 prometheus_client 写一个 exporter
slug: write-an-exporter-using-the-prometheus-client
tags: [ Prometheus, Python ]
date: 2023-02-07T21:37:03+08:00
---

在使用filebeat收集系统日志时，有些网络设备的日志是filebeat通过UDP端口收集的，并且有多个filebeat在使用多个UDP端口同时运行，为了保证日志的完整性，为了避免filebeat意外停止。于是需要监控filebeat的运行状态，首先想到的是[process_exporter](https://github.com/ncabatoff/process-exporter),在调研了proces_exporter的功能后，发现process_exporter对监控同名的多个进程也很麻烦，不能够很好的解决问题。另外一个方案使用blackbox_exporter监控filebeat使用的端口，但是详细了解后发现blackbox_exporter并不支持监控UDP端口，于是开始考虑自己写一个。<!--more-->

# **键盘拿来**

[prometheus_client](https://github.com/prometheus/client_python)  The official Python client for [Prometheus](https://prometheus.io/)，安装和使用详见[官方文档](https://pub.dev/documentation/prometheus_client/latest/prometheus_client/prometheus_client-library.html)，下面贴上我的代码

```python
import os
from flask import Flask, Response, redirect
from prometheus_client import generate_latest, CollectorRegistry, Gauge

app = Flask(__name__)
registry = CollectorRegistry(auto_describe=False)
'''创建一个仓库装收集到的指标'''


@app.route('/metrics')
def hello():
    port_list = [6984, 6985, 6998, 7878, 7879, 7880, 618, 8814, 8815, 8816, 8817, 8818, 8819, 718, 815]
    for i in port_list:
        shell = "netstat -unlp |awk -F: '{print $4}'|grep ^%s |wc -l" % i
        command = os.popen(shell)
        command = command.read()
        gauge = Gauge('Port{}'.format(i), 'show filebeat UDP port status', ['Port'], registry=registry)
        gauge.labels('{}'.format(i)).set(float(command))
    return Response(generate_latest(registry), mimetype='text/plain')


@app.route('/')
def index():
    return redirect("/metrics")
    '''根路由重定向到/metrics'''


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=9117)
```

以上是第一版的代码，使用了flask，prometheus_client,os，为了实现下次新增端口时不需要大改代码，将所有的端口放进了列表，并且有一个致命的问题，就是不能刷新指标界面，一旦刷新就重新触发路由，重新给gauge赋值，这时候就会报错`Duplicated timeseries in CollectorRegistry: {'Port6984'}`意思是这个CollectorRegistry里面有重复的时间序列，因此考虑过怎样把这个CollectorRegistry清空呢？考虑到前面有个注册的步骤，我就尝试用`REGISTRY.unregister`取消注册，几番Google之后还是没能实现，后来查看[官方文档](https://pub.dev/documentation/prometheus_client/latest/prometheus_client/prometheus_client-library.html)，发现Gauge对象有个clear的方法能清除，但是使用起来的效果也不是我想要的，后来便放弃只能另辟蹊径，定时将flask脚本重启，这样Prometheus在采集时就能够获取到最新的数据。

将flask项目配置成service，使用systemd来管理，这样重启比较方便，不需要自己kil，再启动。

```bash
[root@filebeat ~]# cat /usr/lib/systemd/system/flask_exporter.service
[Unit]
Description=flask_exporter.server
After=network.target

[Service]

ExecStart=/usr/bin/python3 /etc/flask_exporter.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

使用crontab重启这个service，问题是crontab的最小粒度是每分钟执行一次，Google到了一个思路如下

```bash
   * * * * * systemctl restart flask_exporter.service
   * * * * * sleep 10; systemctl restart flask_exporter.service
   * * * * * sleep 20; systemctl restart flask_exporter.service
   * * * * * sleep 30; systemctl restart flask_exporter.service
   * * * * * sleep 40; systemctl restart flask_exporter.service
   * * * * * sleep 50; systemctl restart flask_exporter.service
```

总算是把这个问题用各种歪门邪道给解决了，折腾了好久。

# **还能优化**

后来在看别人的代码时发现了新的用法，再次发起了尝试。代码修改如下

```python
import os
import prometheus_client
from prometheus_client import start_http_server, Gauge

prometheus_client.REGISTRY.unregister(prometheus_client.PROCESS_COLLECTOR)
prometheus_client.REGISTRY.unregister(prometheus_client.PLATFORM_COLLECTOR)
prometheus_client.REGISTRY.unregister(prometheus_client.GC_COLLECTOR)
'''取消注册prometheus_client默认会收集的没啥用的数据，这样指标页面就没这些垃圾了。'''
UDP_PORT = Gauge("PORT", "show port status", ['label'])
'''创建一个我们自己的指标，类型是Gauge'''
if __name__ == '__main__':
    start_http_server(9117)
    '''开启http暴露指标，端口在9117'''
    while True:
        port_list = [6984, 6985, 6998, 7878, 7879, 7880, 618, 8814, 8815, 8816, 8817, 8818, 8819, 718, 815]
        for i in port_list:
            shell = "netstat -unlp |awk -F: '{print $4}'|grep ^%s |wc -l" % i
            '''在脚本所在的机器上执行如下命令，如果端口存活，wc的结果就是1，其他任何值都是不正常的。'''
            command = os.popen(shell)
            command = command.read()
            '''获取shell的执行结果'''
            UDP_PORT.labels("{}".format(i)).set(int(command))
            '''将i的值传递给这个UDP_PORT的label，并将command的值转换后赋值给UDP_PORT这个指标'''
```

暴露指标的方式由flask改成prometheus_client自带的`start_http_server`，尽量减少引入外部的模块。

![](https://s2.loli.net/2023/02/02/3nHbPzXLIvtAear.png)

这样解决了指标页面不能刷新的问题，因为根本没用CollectorRegistry，并且指标的名字是一致的（这样在编写告警规则时可以写成`UDP_PORT{}!=0`这样一条规则就能够匹配全部的端口告警，不用为每个端口写一条规则),之前一个Registry里面不允许出现重复的metric。
