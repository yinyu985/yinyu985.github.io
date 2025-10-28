---
title: Grafana 和两辆边车 - sidecar
slug: grafana-and-two-sidecars
tags: [K8s, Docker, Grafana]
date: 2023-03-17T20:50:20+08:00
---

> 在 Kubernetes 中，Sidecar 是一种部署模式，它可以在同一个 Pod 中运行多个容器，其中一个是主容器，其他的容器则是 Sidecar 容器，用来提供一些辅助功能。
>
> 常见的 Sidecar 使用场景包括：
>
> - 日志收集：在一个 Pod 中运行一个主应用程序和一个日志收集器 Sidecar，通过共享 Pod 内的数据卷，让日志收集器能够收集主应用程序产生的日志信息。
> - 数据同步：在一个 Pod 中运行一个主应用程序和一个数据同步器 Sidecar，数据同步器可以将主应用程序产生的数据同步到其他地方（如外部存储或者其他 Pod）。
> - 健康检查：在一个 Pod 中运行一个主应用程序和一个健康检查 Sidecar，通过检查主应用程序的状态，来保证应用程序的可用性和稳定性。
>
> 在 Kubernetes 中，可以通过在同一个 Pod 中定义多个容器来实现 Sidecar 的部署模式。每个容器都可以访问 Pod 的共享网络和存储，从而实现数据的共享和交互。需要注意的是，不同容器之间的生命周期是独立的，它们可以独立启动、停止和重启。

<!--more-->

```bash
> kubectl get pod
NAME                                                     READY   STATUS    RESTARTS        AGE
alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   3 (4m47s ago)   2d17h
prometheus-grafana-656c669c85-49r55                      3/3     Running   3 (4m47s ago)   2d17h
prometheus-kube-prometheus-operator-57674644fc-qd9zm     1/1     Running   1 (4m47s ago)   2d17h
prometheus-kube-state-metrics-745879484f-rpl62           1/1     Running   1 (4m47s ago)   2d17h
prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   2 (4m47s ago)   2d17h
prometheus-prometheus-node-exporter-2mnlv                1/1     Running   1 (4m47s ago)   2d17h

/Users/user.^_^.[10:55:06]
> kubectl get rs
NAME                                             DESIRED   CURRENT   READY   AGE
prometheus-grafana-656c669c85                    1         1         1       2d17h
prometheus-kube-prometheus-operator-57674644fc   1         1         1       2d17h
prometheus-kube-state-metrics-745879484f         1         1         1       2d17h
```

### 疑问：这 grafana 的 rs 里面的 desired 不是一个吗？为啥 pod 的 ready 有三个呢

群里高人指点，这里的三个并不是指数是三个，而是显示这个 pod 里面的三个容器都是 ready 的，并且在 running 状态。

之所以会有三个容器，可能是这个 pod 里面有 init 的容器，或者 sidecar 的容器。你可以通过 describe 查看详细的信息。

下面是 `kubectl describe pod prometheus-grafana-656c669c85-49r55` 的部分输出：

```yaml
Containers:
  grafana-sc-dashboard:
    Container ID:   docker://8e5d3339b69445394c70ecd5b1b1a2009ff7d71fa835019d181eccd434931648
    Image:          quay.io/kiwigrid/k8s-sidecar:1.22.0
    Image ID:       docker-pullable://quay.io/kiwigrid/k8s-sidecar@sha256:eaa478cdd0b8e1be7a4813bc1b01948b838e2feaa6d999e60c997dc823013824
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Fri, 17 Mar 2023 10:50:39 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Tue, 14 Mar 2023 17:20:31 +0800
      Finished:     Fri, 17 Mar 2023 10:50:19 +0800
    Ready:          True
    Restart Count:  1
    Environment:
      METHOD:        WATCH
      LABEL:         grafana_dashboard
      LABEL_VALUE:   1
      FOLDER:        /tmp/dashboards
      RESOURCE:      both
      REQ_USERNAME:  <set to the key 'admin-user' in secret 'prometheus-grafana'>      Optional: false
      REQ_PASSWORD:  <set to the key 'admin-password' in secret 'prometheus-grafana'>  Optional: false
      REQ_URL:       http://localhost:3000/api/admin/provisioning/dashboards/reload
      REQ_METHOD:    POST
    Mounts:
      /tmp/dashboards from sc-dashboard-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-zf92v (ro)
  grafana-sc-datasources:
    Container ID:   docker://e238e224b843a386e9c38f71cf3a9e7c8854413088a6b03b25b15ba3e7d56790
    Image:          quay.io/kiwigrid/k8s-sidecar:1.22.0
    Image ID:       docker-pullable://quay.io/kiwigrid/k8s-sidecar@sha256:eaa478cdd0b8e1be7a4813bc1b01948b838e2feaa6d999e60c997dc823013824
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Fri, 17 Mar 2023 10:50:39 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Tue, 14 Mar 2023 17:20:31 +0800
      Finished:     Fri, 17 Mar 2023 10:50:19 +0800
    Ready:          True
    Restart Count:  1
    Environment:
      METHOD:        WATCH
      LABEL:         grafana_datasource
      LABEL_VALUE:   1
      FOLDER:        /etc/grafana/provisioning/datasources
      RESOURCE:      both
      REQ_USERNAME:  <set to the key 'admin-user' in secret 'prometheus-grafana'>      Optional: false
      REQ_PASSWORD:  <set to the key 'admin-password' in secret 'prometheus-grafana'>  Optional: false
      REQ_URL:       http://localhost:3000/api/admin/provisioning/datasources/reload
      REQ_METHOD:    POST
    Mounts:
      /etc/grafana/provisioning/datasources from sc-datasources-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-zf92v (ro)
  grafana:
    Container ID:   docker://28cfae89d966a86d9deb73fb36bf7fd5784b972b8115067b1baa63aab327bc36
    Image:          grafana/grafana:9.3.8
    Image ID:       docker-pullable://grafana/grafana@sha256:a49f7d74630f47507e7e1ba92f6204f3c7b525d17108a90d489294030a9d507a
    Ports:          3000/TCP, 9094/TCP, 9094/UDP
    Host Ports:     0/TCP, 0/TCP, 0/UDP
    State:          Running
      Started:      Fri, 17 Mar 2023 10:50:39 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Tue, 14 Mar 2023 17:22:09 +0800
      Finished:     Fri, 17 Mar 2023 10:50:19 +0800
    Ready:          True
    Restart Count:  1
    Liveness:       http-get http://:3000/api/health delay=60s timeout=30s period=10s #success=1 #failure=10
    Readiness:      http-get http://:3000/api/health delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:
      POD_IP:                       (v1:status.podIP)
      GF_SECURITY_ADMIN_USER:      <set to the key 'admin-user' in secret 'prometheus-grafana'>      Optional: false
      GF_SECURITY_ADMIN_PASSWORD:  <set to the key 'admin-password' in secret 'prometheus-grafana'>  Optional: false
      GF_PATHS_DATA:               /var/lib/grafana/
      GF_PATHS_LOGS:               /var/log/grafana
      GF_PATHS_PLUGINS:            /var/lib/grafana/plugins
      GF_PATHS_PROVISIONING:       /etc/grafana/provisioning
    Mounts:
      /etc/grafana/grafana.ini from config (rw,path="grafana.ini")
      /etc/grafana/provisioning/dashboards/sc-dashboardproviders.yaml from sc-dashboard-provider (rw,path="provider.yaml")
      /etc/grafana/provisioning/datasources from sc-datasources-volume (rw)
      /tmp/dashboards from sc-dashboard-volume (rw)
      /var/lib/grafana from storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-zf92v (ro)
```

可以看到是有两个 sidecar 的容器的，并且容器的名字也是很符合我的命名规范，也就是见名知义 `grafana-sc-datasources`，意思是 grafana 的 pod 的 sidecar 容器，作用是用于挂载 grafana 的 datasource，一个名字，直接把我一开始的疑问都给解开了。

~~想起来同事写的 Python 代码，我问他为什么这个 instance 后面要写一个 17 呢？他给我的回答是 "因为我喜欢 17 这个数字"，我看你就是喜欢未成年的！！！~~

就是说为什么 grafana 的 pod 里面为什么会有三个容器呢，一个是 sidecar 挂载 datasource，一个是 sidecar 挂载 dashboard，有了这两个，才能让你在第一次打开 grafana 时，数据源和看板都是配置好了的。

本文引用部分全部引用自 chatGPT。

> `quay.io/kiwigrid/k8s-sidecar:1.22.0` 是一个 Kubernetes Sidecar 容器的镜像，主要用于在 Kubernetes 中实现应用程序的监控和管理。
>
> 这个镜像是由 kiwigrid 团队开发的，其主要特点包括：
>
> - 基于 Alpine Linux 3.14.1 构建，镜像大小仅为 8.82 MB。
> - 支持多种监控方式，包括 Prometheus、Grafana 和 InfluxDB 等。
> - 可以轻松地与 Kubernetes 网络和存储系统进行集成。
> - 支持动态配置，允许在不停止应用程序的情况下更新配置。
> - 可以在应用程序容器旁边作为 Sidecar 容器运行，实现应用程序的监控和管理。
>
> 这个镜像主要用于实现 Kubernetes 中的 Sidecar 模式，即将一个或多个辅助容器部署在主应用程序容器旁边，以协同工作或提供额外的功能。Sidecar 模式可以简化应用程序的监控和管理，同时提高应用程序的可靠性和可维护性。

不是真的不知道 sidecar 是啥，而是在遇到问题、解决问题的路上发现，哦，原来是这小子。之所以本文标题取这个名字，是因为当你使用翻译软件翻译英文文档时，sidecar 真的就会被翻译成“边车”，哈哈哈，很逗，但也真不知道有啥更好的选择了。