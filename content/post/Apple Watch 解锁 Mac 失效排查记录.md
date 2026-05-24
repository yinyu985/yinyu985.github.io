---
title: Apple Watch 解锁 Mac 失效排查记录
slug: apple-watch-unlock-mac-auto-unlock-stale-credential
tags: [Mac, Apple Watch, Troubleshooting]
date: 2026-05-24T22:30:00+08:00
---

今天又碰到一个很苹果的问题：系统设置里明明已经打开了“使用 Apple Watch 解锁应用程序和 Mac”，但实际锁屏以后就是不能解锁。

更恶心的是，iPhone 和 Apple Watch 之间互相解锁是正常的，Apple Watch 也能弹出确认，用来批准钥匙串访问、系统授权这类操作。也就是说，表不是坏的，Apple ID 也不是完全不通，蓝牙和 Wi-Fi 也不是简单没开。<!--more-->

最后查下来，问题不在 Shadowrocket，不在 Tailscale，也不在局域网本身，而是 Mac 本地的 Auto Unlock 凭据和配对状态陈旧了。

苹果这个功能已经出了很多年，网上几年前的老帖子到今天还管用，说明这套状态机还是会卡死，而且 UI 上看不出来。

## 现场现象

机器是 Mac mini M4，系统是 macOS 15.7.3。

系统设置里这个开关是打开的：

```text
登录密码 -> Apple Watch -> 使用 Apple Watch 解锁应用程序和 Mac
```

蓝牙能扫到手表：

```text
Y的Apple Watch
RSSI: -56 ~ -61
```

Wi-Fi 也正常：

```text
Wi-Fi: en1
IP: 192.168.31.184
awdl0: active
```

`awdl0` 是 AirDrop、Continuity、Auto Unlock 这类功能会用到的近场链路。它是 active，说明不是那种一眼能看出来的无线链路失效。

还有一个很关键的现象：打开钥匙串访问时，Apple Watch 上能弹出确认。这说明 Mac 到 Watch 的近场认证通道是活的。

所以问题不是“Apple Watch 完全不能跟 Mac 通信”，而是“登录窗口自动解锁 Mac”这一套单独坏了。

## 最关键的异常

真正有价值的是这个命令：

```bash
defaults read com.apple.sharingd AutoUnlockLastSeenWatchDate
```

当时看到的是：

```text
2026-01-29 11:48:24 +0000
```

换成本地时间是 2026-01-29 19:48:24。

但实际排查日期是 2026-05-24。也就是说，系统设置 UI 虽然显示开关打开，`sharingd` 底层却认为最近一次见到可用于 Auto Unlock 的 Apple Watch 还是几个月前。

这基本就说明 UI 状态和底层 Auto Unlock 状态已经不一致了。

再看完整一点：

```bash
defaults read com.apple.sharingd
```

里面有几项很关键：

```text
AutoUnlockWatchCurrentlyInList = 1;
AutoUnlockWatchExistedInUnlockList = 1;
AutoUnlockLastSeenWatchDate = "2026-01-29 11:48:24 +0000";
AutoUnlockDevicesMissingSessionKey = ();
```

这表示系统知道这块表在解锁列表里，但最近可用状态没有刷新。

## 不要被网络方向带偏

我本地确实长期运行 Shadowrocket，也安装了 Tailscale。

当时网络状态里能看到：

```text
Shadowrocket: connected
Tailscale: connected
default route: utun
```

这类 VPN、Packet Tunnel、Network Extension 确实会增加变量，尤其会影响 Apple ID、IDS、iCloud 相关服务。但这次它不是根因。

判断依据很简单：

- Apple Watch 能确认钥匙串访问
- Mac 能通过蓝牙看到 Watch
- Wi-Fi 和 AWDL 都正常
- Auto Unlock 的 last seen 时间停在几个月前
- 钥匙串里存在陈旧的 Auto Unlock 凭据

所以这次不要优先按网络问题处理。尤其在中国大陆环境里，Shadowrocket 可能是正常联网的前提，不能随便断。

## 找到陈旧的钥匙串凭据

继续查钥匙串：

```bash
security find-generic-password -a com.apple.continuity.auto-unlock.attested ~/Library/Keychains/login.keychain-db
```

能看到一条隐藏项：

```text
Auto Unlock: yy的Mac mini
acct = com.apple.continuity.auto-unlock.attested
created = 2026-02-07
modified = 2026-02-07
```

这条就是本机 Auto Unlock 的认证凭据。

问题在于，这条记录也不是今天新建的，而是很早之前留下的。结合 `AutoUnlockLastSeenWatchDate` 停在 1 月底，基本可以判断：Mac 拿着一套旧的 Auto Unlock 状态，但 UI 仍然显示功能已开启。

这就是最坑的地方：UI 开关是绿色，不代表底层凭据健康。

## 网上老方案为什么仍然有效

网上有一个几年前的方案，大概步骤是：

1. 打开钥匙串访问
2. 显示隐藏项目
3. 搜索 `Auto Unlock`
4. 删除相关记录
5. 搜索 `AutoUnlock`
6. 删除 `tlk`、`tlk-nonsync`、`classA`、`classC`
7. 删除 `~/Library/Sharing/AutoUnlock/` 里的 `ltk.plist` 和 `pairing-records.plist`
8. 回到系统设置重新开启 Apple Watch 解锁，第一次可能失败，第二次再开

这个方案看起来很古老，但它的本质是对的：清掉 Mac 本地 Auto Unlock 的旧凭据和旧配对记录，让系统重新建链。

这次最终也是这个方向解决的。

## 这次有效的恢复流程

最后有效的步骤是：

```text
关闭 Mac 上 Apple Watch 解锁开关
清理 Auto Unlock 相关钥匙串/配对残留
重启 Mac
重新打开 Apple Watch 解锁开关
第一次开启失败
第二次开启成功
锁屏测试，Apple Watch 成功解锁 Mac
```

其中有两个细节值得记住。

第一，第一次开启失败不一定说明方案没用。这个功能重建凭据时，第一次失败、第二次成功是网上很多人都遇到过的现象。

第二，`~/Library/Sharing/AutoUnlock/` 要看当前用户的家目录，不是 root 的家目录。

如果用 root shell 执行：

```bash
ls ~/Library/Sharing/AutoUnlock/
```

实际看的是：

```text
/var/root/Library/Sharing/AutoUnlock/
```

这当然可能不存在。正确路径应该是：

```bash
/Users/yy/Library/Sharing/AutoUnlock/
```

## 下次复发时的快速判断

先不要重装系统，也不要先怀疑路由器，更不要先把 VPN 断掉。

先看这几个状态：

```bash
defaults read com.apple.sharingd AutoUnlockLastSeenWatchDate
defaults read com.apple.sharingd AutoUnlockWatchCurrentlyInList
defaults read com.apple.sharingd AutoUnlockDevicesMissingSessionKey
```

再看钥匙串里是否有旧的 Auto Unlock 凭据：

```bash
security find-generic-password -a com.apple.continuity.auto-unlock.attested ~/Library/Keychains/login.keychain-db
```

如果 `AutoUnlockLastSeenWatchDate` 明显不是最近时间，并且钥匙串里存在旧的 `Auto Unlock:` 项，就直接按凭据重建方向处理。

## 推荐的处理顺序

下次再出现这个问题，我会按这个顺序来：

1. 确认 Apple Watch 戴在手腕上，并且已解锁
2. 确认 iPhone 和 Watch 互通正常
3. 确认 Watch 能否批准钥匙串访问或系统授权
4. 查看 `AutoUnlockLastSeenWatchDate`
5. 如果时间陈旧，关闭 Apple Watch 解锁开关
6. 清理 Auto Unlock 相关钥匙串项
7. 必要时清理 `~/Library/Sharing/AutoUnlock/` 下的配对文件
8. 重启 Mac 和 Apple Watch
9. 重新打开 Apple Watch 解锁开关
10. 如果第一次失败，等几秒再打开第二次

这比反复开关 Wi-Fi、蓝牙、VPN 靠谱得多。

## 结论

这次的根因可以概括成一句话：

> Apple Watch 和 Mac 的基础通信没坏，坏的是 Mac 本地 Auto Unlock 的陈旧凭据和配对状态。

苹果的问题在于，系统设置 UI 只告诉你“开关是开的”，但不会告诉你底层 Auto Unlock 会话已经死了几个月。

所以以后再遇到这个问题，不要被绿色开关骗了。先看 `AutoUnlockLastSeenWatchDate`，再看钥匙串里的 `com.apple.continuity.auto-unlock.attested`。

如果它们都很陈旧，那就别继续玄学重启了，直接清理凭据、重建 Auto Unlock。
