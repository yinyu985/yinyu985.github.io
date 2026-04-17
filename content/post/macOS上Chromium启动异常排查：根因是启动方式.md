---
title: macOS 上 Chromium 启动异常排查：根因是启动方式
slug: chromium-browser-startup-issue-on-macos
tags: [Mac, Browser]
date: 2026-04-17T09:00:00+08:00
---

我在 macOS 上一直用一条 alias 启动 Chrome，用来绕过公司网络里一些自签证书场景：

```bash
alias ge='nohup "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --ignore-certificate-errors --ignore-urlfetcher-cert-requests &>/dev/null &'
```

这条命令以前一直能用，最近却突然开始报错：

> 系统无法读取您的偏好设置。某些功能可能无法使用，并且对偏好设置所做的更改不会保存。

第一反应很自然：是不是 profile 坏了，或者权限被系统/安全软件动过。实际排下来，根因不在这里。<!--more-->

## 先看表象

表面症状很像 profile 异常：

- Chrome 启动后弹出偏好设置无法读取的提示
- 原有插件看起来像没加载
- 依赖插件的代理切换失效
- Brave 也出现了类似现象

我先确认了 Chrome 正在使用的 profile 路径：

```text
/Users/yinyu/Library/Application Support/Google/Chrome/Default
```

这说明它并没有跑到一个奇怪的新目录里，问题不太像“启动错 profile”。

## 为什么一度怀疑是权限问题

我试着直接访问目录，结果在 Terminal 里会报：

```bash
ls: fts_read: Operation not permitted
```

`sudo` 也一样不行。再看目录属性，属主、ACL、flag 都没有明显异常。Finder 反而能打开这个目录，这些现象很容易让人继续怀疑是 macOS 权限或者 TCC 问题。

我还看了 `xattr`，目录上有：

```bash
com.apple.provenance:
com.apple.quarantine: 0081;00000000;Chrome;
```

一度以为是 quarantine 把目录锁住了，但尝试清理以后还是不行。也就是说，目录层面的这些异常看着很像根因，最后却都不是主线。

## 真正的问题

真正值得怀疑的是启动方式本身。

我原来的 alias 干了几件事：

1. 直接执行 `.app` bundle 里的二进制
2. 用 `nohup` 脱离终端
3. 把所有输出都丢掉
4. 再附带两个 Chromium 的证书相关参数

这不是 macOS 上标准的 GUI 启动链路。它更像把浏览器当成普通 CLI 程序在跑。

在以前的版本上，这种方式可能“碰巧能用”；但 Chromium 系浏览器不断升级后，启动上下文、profile 初始化、sandbox、扩展加载这些环节都更敏感了，旧的 hack 就开始不稳定。

## 重新验证

我把启动方式改成了 macOS 推荐的 `open`：

```bash
open -na "Google Chrome" --args --ignore-certificate-errors --ignore-urlfetcher-cert-requests
```

改完以后问题立刻消失了。Brave 也可以用同样的方式启动，现象一致，说明根因更像是 Chromium 系浏览器对这种老启动方式不再稳定支持，而不是某个单独 profile 被损坏。

## 结论

这次排查最容易误导人的地方，是“无法读取偏好设置”这个提示太像 profile 出问题了。但实际上，问题出在我长期依赖的一条老 alias：

```text
直接执行 Chromium app 内部二进制 + nohup + 吞日志
```

在 macOS 上，GUI App 最稳妥的方式还是：

```bash
open -a "Google Chrome" --args ...
```

而不是把 `.app` 里的可执行文件当普通程序直接跑。对 Chromium 系浏览器来说，这种差异以前可能不明显，现在已经足够影响启动稳定性了。

## 最终可用配置

```bash
alias ge='open -na "Google Chrome" --args --ignore-certificate-errors --ignore-urlfetcher-cert-requests'
alias be='open -na "Brave Browser" --args --ignore-certificate-errors --ignore-urlfetcher-cert-requests'
```

如果你也在 macOS 上用类似的 alias 启动浏览器，遇到莫名其妙的 profile 报错，可以先别急着修权限，先检查一下自己是不是把 GUI App 当 CLI 程序在跑了。
