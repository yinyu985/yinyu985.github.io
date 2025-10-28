---
title: Kuboard-spray 图形化安装高可用的 Kubernetes
slug: graphical-installation-of-highly-available-kubernetes-for-kuboard-spray
tags: [K8s, Tools, Docker]
date: 2023-07-07T18:02:02+08:00
---

不管你有没有听说过 [Kuboard](https://kuboard.cn/)，它都是一个非常有名的 K8s 管理工具，官方描述的是 `Kuboard - Kubernetes 多集群管理界面`，我这次想介绍的是 `Kuboard-spray`<!--more-->

如题，安装高可用的 Kubernetes，先介绍下 Kubespray，它是 Kubernetes SIG（不是官方，SIG 的意思是特别兴趣小组）提供的 Kubeadm 之外的一种安装方式，仓库里描述的是：

> Deploy a Production Ready Kubernetes Cluster

部署一个生产可用的 Kubernetes 集群。经过我的了解，这个确实比较专业，自带 [CIS](https://www.ibm.com/cn-zh/topics/cis-benchmarks) 扫描，为 Kubernetes 部署和配置提供一组 Ansible 角色。Kubespray 可使用 AWS、GCE、Azure、OpenStack 或裸机基础架构即服务（IaaS）平台。Kubespray 是一个开源项目，具有开放的开发模型。对于已经了解 Ansible 的人来说，该工具是一个不错的选择，因为不需要使用其他工具进行配置和编排。

很牛，但是我不用，我用 Kuboard-spray 图形化界面更香。下面简单介绍下，需要使用自己去看[官方文档](https://kuboard.cn/install/install-k8s.html#kuboard-spray)：

1. 在准备好安装 K8s 的机器后，需要额外的一台机器运行 Kuboard-spray
2. 使用 Docker 安装运行 Kuboard-spray 即可
3. Docker 启动成功后，访问 Web 界面即可开始配置 Kubespray
4. 添加你要部署 K8s 的机器，设定角色，是否为 etcd 节点等，设定 P，配置密码
5. 配置软件包管理器的源，指定你安装 K8s 的机器的类型，指定网络插件
6. 你可以自定义是否安装 Kuboard，毕竟是师出同门，肯定要推广一下
7. 最后一定要检查配置，毕竟不是过家家，万一重来就是半个小时

回忆我的两次安装经历都是十分顺利，没有遇到任何问题。装完之后，Kuboard-spray 还能连接集群、备份 etcd、获取 kubeconfig，或者来一个 CIS 扫描，也能连接到机器执行命令。总之就是很强大啊，值得分享。