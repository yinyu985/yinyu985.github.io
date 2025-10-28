---
title: Minikube 中使用 Helm 安装 Prometheus 监控大全套
slug: using-helm-to-install-prometheus-monitoring-complete-set-in-minikube
tags: [Linux, Prometheus, Docker, Minikube]
date: 2023-03-15T20:46:44+08:00
---

Minikube is a tool that allows you to run a Kubernetes cluster on your local machine. It is designed to make it easy to develop and test applications that will be deployed to a production Kubernetes environment. Minikube runs a single-node Kubernetes cluster inside a virtual machine on your local machine, which allows you to simulate a real-world Kubernetes environment without the need for additional hardware.<!--more-->

Minikube 的安装在[官方文档](https://minikube.sigs.k8s.io/docs/start/)有详细介绍，此处不再赘述。[本文](https://k21academy.com/docker-kubernetes/prometheus-grafana-monitoring/)主要记录在 Minikube 中使用 Helm 安装 Prometheus 监控大全套。

## Helm

> Helm 是一个用于 Kubernetes 应用程序部署和管理的包管理器。它的主要作用是管理 Kubernetes 中的 Charts，它们是一组预定义的 Kubernetes 资源模板，可以在 Kubernetes 群集中部署。Helm 可以让您轻松地创建、共享和安装基于 Kubernetes 的应用程序，以及管理它们的版本和依赖关系。

以下是 Helm 的一些核心概念：

- Chart：一个 Chart 包含了一组预定义的 Kubernetes 资源模板，它们可以一起部署一个应用程序。一个 Chart 可以包含多个 YAML 文件，用于定义部署、服务、配置等资源。Chart 通常包含一个 `Chart.yaml` 文件和一个 `values.yaml` 文件，分别定义了 Chart 的元数据和默认配置。
- Repository：Helm Chart 存储库是一个可公开访问的位置，其中包含一组可用的 Charts。存储库通常使用 HTTP 服务器托管，允许用户从远程访问和下载 Charts。
- Release：在 Helm 中，Release 是指一个 Chart 的实例。在部署过程中，Helm 将 Chart 渲染为 Kubernetes 资源，并将其安装到群集中。每个 Release 都有一个唯一的名称，允许您在部署多个版本的相同 Chart 时对它们进行区分。
- Values：Values 是 Chart 中的一组默认配置，用于控制应用程序的部署和行为。它们可以在安装 Chart 时通过命令行标志或 YAML 文件进行覆盖，以定制化 Chart 的部署。
- Template：模板是 Chart 中的 Kubernetes 资源模板，它们可以通过 Go 中的文本模板语言进行定义。Helm 使用模板来根据 Values 生成 Kubernetes 资源，并将其部署到 Kubernetes 群集中。

使用 Helm，您可以轻松地创建、打包和共享 Chart，以及管理它们的版本和依赖关系。您还可以使用 Helm 在 Kubernetes 群集中安装、升级和回滚应用程序，以及在多个群集之间共享 Charts。

## Prometheus

> Prometheus 监控系统是一种开源的监控解决方案，可用于监控云计算、容器和微服务环境中的应用程序。它最初由 SoundCloud 开发，现在由 CNCF（Cloud Native Computing Foundation）管理。
>
> Prometheus 使用 Pull 模型来收集指标，这意味着它从要监控的应用程序中拉取指标数据。它支持许多不同类型的指标，包括计数器、测量值和摘要。Prometheus 使用 PromQL（Prometheus Query Language）查询语言来查询和聚合指标数据，从而使用户能够对监控数据进行更深入的分析和可视化。
>
> Prometheus 具有许多高级功能，例如自动发现服务、动态配置、报警和可视化。它还具有广泛的集成能力，可与其他工具和系统集成，如 Grafana、Alertmanager 和 Kubernetes。
>
> 总的来说，Prometheus 是一种功能强大、灵活和易于使用的监控解决方案，可帮助开发人员和运维人员更好地理解和管理他们的应用程序和基础设施。

对于一个已经成功安装配置的 Minikube，我们还需要安装 Helm。在 macOS 中，使用 brew 作为包管理器安装 Helm 将会非常方便，只需一条命令：

```bash
brew install helm
```

安装成功后，执行 `helm version` 会显示当前 Helm 的版本。目前是 v3，网上的其他教程在使用 Helm 时，无脑复制命令可能会报错，因为 Helm 新版本抛弃了一些命令参数。~~在此插一句题外话，云原生的东西，更新快的很，Prometheus、Alertmanager、Grafana，能容器化就别二进制，真的很傻逼。等你某天发现了一个特性，然后想用这，但是发现你二进制安装的东西不允许你随随便便升级时，你会想起我说的话。~~

```bash
/Users/user.^_^.[10:59:03]
> helm version
version.BuildInfo{Version:"v3.10.2", GitCommit:"50f003e5ee8704ec937a756c646870227d7c8b58", GitTreeState:"clean", GoVersion:"go1.19.3"}
```

上面提到了 Helm 的基础概念，现在我们需要像为 yum 配置仓库那样，为 Helm 添加仓库，添加完后再执行更新：

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update
```

见证奇迹的时候到了：

```bash
helm install prometheus prometheus-community/kube-prometheus-stack
```

就这一条命令，Helm 就帮你把 Prometheus 大全套，包括 Prometheus、Grafana、Alertmanager、node_exporter 全都装进了 Minikube。

你可以新开一个终端窗口（我使用的是 iTerm2，high level），之所以需要新开，因为这个命令是在前台执行的：

```bash
minikube dashboard
```

Minikube 将会启动一个跟 k8s 一模一样的 dashboard，并且自动在浏览器打开。在这里你能看到 Minikube 当前运行的 deployment、service、pod 等各类信息。

不过我用的是 Lens，一个个人用户免费使用的 k8s 的 IDE，在 Lens 能够很方便地查看 k8s（Minikube）中的资源状态，能提供方便的编辑方式。

已经通过 Helm 安装了 Grafana，我们想上去看看呢？我可以告诉你：通过 Helm，Helm 使用 prometheus-operator 安装的 Grafana 的默认用户名和密码分别是 `admin` / `prom-operator`。当然，如果你很有耐心地在 Minikube dashboard 的界面点上几分钟，分别看看里面都有啥，你会在 Secret 的分栏里看到：

![image-20230315112034888](https://s2.loli.net/2023/03/15/zXoRy45Q3Pivbnf.png)

那么知道了 Grafana 的信息，如何访问呢？已知刚才创建的 service 都是 ClusterIP 类型的。

> 在 Kubernetes 中，ClusterIP 是一种用于访问集群内部服务的虚拟 IP 地址。它代表一个 Kubernetes Service，该 Service 会将请求转发到与之关联的 Pod。ClusterIP 仅在集群内部可访问，外部网络无法访问。
>
> 在 Minikube 中，可以使用 `minikube service` 命令来暴露 Kubernetes Service，并将其映射到主机上的随机端口。此时，您可以使用主机 IP 地址和该端口号访问该服务。
>
> 例如，如果您有一个名为 `my-service` 的 Service，您可以使用以下命令将其暴露出来：
>
> ```bash
> minikube service my-service
> ```
>
> 此命令将启动一个本地代理，将 `my-service` 映射到主机上的一个随机端口，并在浏览器中打开该服务。
>
> 如果您想直接使用 ClusterIP 访问该服务，您可以通过以下方式获取 ClusterIP：
>
> ```bash
> kubectl get service my-service
> ```
>
> 该命令将返回 `my-service` 的详细信息，其中包括 ClusterIP。您可以使用该 IP 地址和 Service 暴露的端口号访问该服务。但是需要注意的是，ClusterIP 只能在 Minikube 内部访问，无法从外部网络访问。

ClusterIP 只能在 Minikube 内部访问，无法从外部网络访问，意思就是说，你在 Mac 上访问 ClusterIP 还是不行的。如果你只想通过 ClusterIP 访问的话，Minikube 提供一种方式：通过 `minikube ssh` 进入到 Minikube 模拟出的机器内部——就是 Minikube 中运行的 k8s 也是在这台机器里的。通过获取到的 ClusterIP 和 curl 命令：

```bash
curl http://10.105.58.216:9093
curl http://10.105.91.133:9090
curl http://10.107.64.147:9100/metrics
```

这样访问是好使的。当然，最好的办法还是使用 minikube 转发：

```bash
minikube service prometheus-grafana
|-----------|--------------------|-------------|--------------|
| NAMESPACE |        NAME        | TARGET PORT |     URL      |
|-----------|--------------------|-------------|--------------|
| default   | prometheus-grafana |             | No node port |
|-----------|--------------------|-------------|--------------|
😿  service default/prometheus-grafana has no node port
🏃  Starting tunnel for service prometheus-grafana.
|-----------|--------------------|-------------|------------------------|
| NAMESPACE |        NAME        | TARGET PORT |          URL           |
|-----------|--------------------|-------------|------------------------|
| default   | prometheus-grafana |             | http://127.0.0.1:52850 |
|-----------|--------------------|-------------|------------------------|
🎉  正通过默认浏览器打开服务 default/prometheus-grafana...
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

Minikube 开启了一个 tunnel，将 ClusterIP 和端口转发到了 Mac 本地的 `http://127.0.0.1:52850`。如果你使用 Lens 这样的工具，能够在 Service 分栏里配置一个端口转发（Port Forwarding），这样能够实现在 Mac 访问 Minikube 中的 ClusterIP 的效果。

另外，使用 `kubectl` 也能够实现端口转发，我想 Lens 应该也是将点点点的操作转换成了 `kubectl` 这样的命令来实现端口转发的。

```bash
kubectl port-forward deployment/prometheus-grafana 3000
```