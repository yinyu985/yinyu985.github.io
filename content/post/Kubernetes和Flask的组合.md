---
title: Kubernetes和Flask的组合
slug: combination-of-kubernetes-and-flask
tags: [ K8s, Minikube, Python, Flask ]
date: 2023-05-14T16:27:31+08:00
---

众所周知Minikube有自带的dashboard，输入命令`minikuke dashboard `打开链接就能看到，某日突发奇想，加入公司内部需要一个自定义的kubernetes监控平台，以满足一些自定义的需求呢？比如，我想看到最近新建的100个pod，或者我想看最近的k8s集群的events，使用flask来开发一个平台满足自定义，是一个不错的选择。<!--more-->

```python
from flask import Flask, render_template
from kubernetes import client, config

app = Flask(__name__)

#加载Kubernetes配置和连接Minikube的上下文
conf = config.load_kube_config(context="minikube")

#获取Kubernetes API对象
v1 = client.CoreV1Api()

#查询所有Pod并在前端页面上进行展示
@app.route("/")
def home():
    #查询所有Pod
    pods = v1.list_pod_for_all_namespaces(watch=True)

    #渲染模板并返回HTML响应
    return render_template("index.html", pods=pods.items)

#启动Flask应用
if __name__ == "__main__":
    app.run(debug=True)
```

在template文件夹下面创建html文件

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

运行flask项目，打开前端页面，就能看到Minikube集群中全部的pod都被列了出来，并且我把watch的参数设置成了true，也就是我新建一个pod，然后刷新页面，新的pod也会被列出。牛逼，Python是如何连接Minikube的呢，全靠一句`from kubernetes import client, config`,真就是前人栽树后人乘凉，不用去自己找kubeconfig，不用去找apiserver的地址，直接导入，指定类型为Minikube，搞定！这里只是列出了pod的简单示例，后续还有无限的想象力。
