---
title: Kubernetes 和 Flask 的组合
slug: combination-of-kubernetes-and-flask
tags: [K8s, Minikube, Python, Flask]
date: 2023-05-14T16:27:31+08:00
---

众所周知，Minikube 有自带的 dashboard，输入命令 `minikube dashboard` 打开链接就能看到。某日突发奇想，如果公司内部需要一个自定义的 Kubernetes 监控平台，以满足一些个性化的需求呢？比如，我想看到最近新建的 100 个 Pod，或者我想查看最近的 K8s 集群 Events。使用 Flask 来开发一个平台满足这些自定义需求，是一个不错的选择。<!--more-->

```python
from flask import Flask, render_template
from kubernetes import client, config

app = Flask(__name__)

# 加载 Kubernetes 配置并连接 Minikube 的上下文
conf = config.load_kube_config(context="minikube")

# 获取 Kubernetes API 对象
v1 = client.CoreV1Api()

# 查询所有 Pod 并在前端页面上进行展示
@app.route("/")
def home():
    # 查询所有 Pod
    pods = v1.list_pod_for_all_namespaces(watch=False)

    # 渲染模板并返回 HTML 响应
    return render_template("index.html", pods=pods.items)

# 启动 Flask 应用
if __name__ == "__main__":
    app.run(debug=True)
```

> **注意**：原代码中 `watch=True` 会导致请求长期挂起，不适合用于 Web 页面直接渲染。此处已修正为 `watch=False`，以获取当前状态并立即返回。

在 `templates` 文件夹下创建 HTML 文件：

```html
<!doctype html>
<html>
  <head>
    <title>Kubernetes Pods</title>
  </head>
  <body>
    <h1>Kubernetes Pods:</h1>
    <ul>
      {% for pod in pods %}
        <li>{{ pod.metadata.name }}</li>
      {% endfor %}
    </ul>
  </body>
</html>
```

运行 Flask 项目，打开前端页面，就能看到 Minikube 集群中所有的 Pod 都被列了出来。当我新建一个 Pod 后，刷新页面，新的 Pod 也会出现在列表中。牛逼！Python 是如何连接 Minikube 的呢？全靠一句 `from kubernetes import client, config`，真就是前人栽树后人乘凉——不用自己去找 kubeconfig，也不用手动查找 apiserver 地址，直接导入，指定上下文为 Minikube，搞定！

这里只是展示了 Pod 的简单示例，后续还有无限的想象力。