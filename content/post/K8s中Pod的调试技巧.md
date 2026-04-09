---
title: K8s 中 Pod 的调试技巧
slug: debugging-techniques-for-pod-in-k8s
tags: [K8s, Docker]
date: 2023-06-13T20:53:12+08:00
---

今天在逛大佬的博客时又学到好东西了，特此记录一下，关于 K8s 中 Pod 调试的奇技淫巧，虽然不一定能用到，但是值得记录<!--more-->

## k8s 的 pod exit 非 0 调试

Pod 一直 crash 的话，我们可以把 command 用下面这个，生存探针啥的全部注释掉。command 用 `bash`、`tail` 之类的会在调试完成 delete 或 kill 的时候阻塞住，参考 [infinite blockings](https://stackoverflow.com/questions/2935183/bash-infinite-sleep-infinite-blocking)。

```
["bash", "-c", "coproc { exec >&-; read; }; eval exec \"${COPROC[0]}<&-\"; wait"]
```

## 解读

> 这个 Dockerfile 命令是一种巧妙的方法，用于保持 Pod 运行状态以进行调试。它做了以下几件事：
>
> 1. 使用 `bash -c` 执行一个命令
>
> 2. `coproc { exec >&-; read; }` 在后台启动一个协程，并立即阻塞读取输入
>
> 3. `eval exec "${COPROC[0]}<&-"` 将 shell 的标准输入重定向到该协程
>
> 4. `wait` 等待 shell 命令结束
>
> 这样的效果是：
>
> - 启动一个一直阻塞读取输入的协程
> - 将 shell 的标准输入重定向到那个协程
> - shell 本身会一直 `wait` 等待，维持运行状态
>
> 所以，Pod 中的这个 `bash` 命令会一直运行等待，不会退出。这就保证了 Pod 也不会因为命令退出而退出。这给我们时间在 Pod 内进行调试，而不必担心它会因为命令退出而退出：
>
> - 我们可以在其中运行其他调试命令
> - 可以查看日志、事件等诊断信息
> - 可以使用 `kubectl exec` 进入 Pod 交互调试
>
> 当调试完成，我们可以：
>
> - 删除 Pod 让其退出
> - 发送 `SIGINT` 给 `bash` 进程退出 `wait` 状态，从而退出 Pod
> - `docker kill` 杀死容器
>
> 这种方法相比直接删除 Pod 的优点是：
>
> - 给我们时间在“运行状态”下进行调试
> - Pod 退出后，事件、日志等诊断信息也不会丢失，更有助于排查问题
>
> 所以，总结来说，这是一个巧妙的方法来维持容器运行以进行调试。通过一个一直 `wait` 的 shell 命令，来防止 Pod 因为命令退出而直接退出。