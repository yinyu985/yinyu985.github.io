---
title: 记录在 K8s 中部署 Nginx 并修改主页
slug: record-the-deployment-of-nginx-in-the-middle-of-k8s-and-modify-the-homepage
tags: [Docker, Minikube, Nginx]
date: 2023-06-20T20:56:37+08:00
---

如题，记录在 K8s 中部署 Nginx 并修改主页<!--more-->

经过 Github 的学生认证即可使用 Github Copilot，前段时间获得了 Github Copilot Chat，今天听人说 Github Copilot 还能写 Yaml，不禁有个问题，Yaml 算编程语言吗？我觉得算吧，它比 Json 好啊，可读性强，并且是各种云原生常见应用的配置格式，就当然是云原生编程语言吧。今天就来用 Github Copilot 当一回 Yaml engineer 吧。打开 VS Code Insider（Chat 插件目前是内测功能，所以只能在 Insider 版本的 VS Code 下载安装）。

## 让 Copilot 帮我写一个 Yaml 实现如下功能

> 部署 Nginx 的 Deployment，指定 namespace 为 default，副本数为 1，镜像为 nginx:1.7.9，容器名为 nginx，容器端口为 80，创建 cm 修改 index 文件内容为“你好Minikube!”，因为有中文，再创建一个 cm 修改 Nginx 的配置文件设定 Nginx 的字符集为 `UTF-8`，创建一个 Service，端口为 80，targetPort 为 30080，protocol 为 TCP。

## 记录一下这里的两个坑

### Pod 的生命周期通常是短暂的。它们会启动一个或多个容器来执行任务，一旦任务完成，Pod 就会退出

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:1.7.9
      ports:
        - containerPort: 80
      command: ["/bin/sh"]
      args: ["-c", "echo hello k8s > /usr/share/nginx/html/index.html"]
```

也就是如果 command 里面写了 echo，那么当 echo 命令执行完了，Pod 的任务就完成了，就会退出，日志都看不到。

### 不同的镜像有不同的启动命令，这个可以去看镜像的 Dockerfile，如果你写了 command，就会将其覆盖

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:1.7.9
      ports:
        - containerPort: 80
      command: ["/bin/sh"]
      args: ["-c", "echo hello k8s > /usr/share/nginx/html/index.html; while true; do echo hello k8s; sleep 10; done"]
```

对于 Nginx 容器来说，不需要显式地指定 `command` 字段，因为该字段默认为 `["nginx", "-g", "daemon off;"]`，已经在 Nginx 镜像中设置好了。这个默认的命令会启动 Nginx 进程并保持其在前台运行。如果知道了上面那个情况，然后又写了个 `while true` 来避免容器退出，还是会出现问题的。

> spec.template.spec.containers.command 定义的是 shell 命令，这会使得 Nginx 容器启动一个 shell 进程，而不是 Nginx 进程。所以 Nginx 服务不会正常启动。

对于这种需要修改 index.html 的场景，其实是可以使用 `initContainers`，或者 mount 一个 ConfigMap 到容器来实现修改。

## 记录两种正确的操作

### initContainers

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      initContainers:
        - name: setup
          image: busybox
          command: ["/bin/sh", "-c"]
          args:
            - echo "Running initialization commands..."
            - echo "Hello from initContainer!"
      containers:
        - name: nginx
          image: nginx:1.7.9
          ports:
            - containerPort: 80
```

### configmap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: default
data:
  index.html: |
    你好Minikube!
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
  namespace: default
data:
  default.conf: |
    charset utf-8;
    server {
      listen 80;
      server_name nginx;
      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.7.9
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
            - name: nginx-html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: nginx-config
          configMap:
            name: nginx-conf
        - name: nginx-html
          configMap:
            name: nginx-config
            items:
              - key: index.html
                path: index.html
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

其中第二种方法是我采用的，因为往 index.html 里面写入了中文，于是又多了一个步骤，就是需要再来一个 ConfigMap，在 Nginx 的配置文件中指定字符集为 `UTF-8`。全程让 Copilot 补全，我只需要让他知道我要干什么，然后进行微调，就能投入使用了。不得不说，确实强大，离下岗又近了一步。

最后因为是在 Minikube 环境中部署的，通过 `minikube service nginx-service`，就能在 Mac 本机访问到 Minikube 中的服务了。