---
title: code-server WebSSH 被 macOS TCC 拦住这件事
slug: code-server-webssh-macos-tcc-documents
tags: [Mac, code-server, AI]
date: 2026-05-21T10:20:00+08:00
---

今天碰到一个很离谱的问题：Mac mini 上的 code-server 原本用得好好的，WebSSH 里突然开始各种报错。

最典型的是这两个：

```bash
Error: The current working directory must be readable to yy to run brew.
```

```bash
fatal: Unable to read current working directory: Operation not permitted
```

看起来像是 `brew` 有问题，又像是 Git 仓库坏了，还像是当前目录权限不对。实际最后查下来，根本不是这些。<!--more-->

真正的原因是：code-server 从 brew 安装换成官方脚本安装以后，背后的 Node 路径变了，macOS TCC 对新路径没有放行，甚至明确拒绝了它访问 `~/Documents`。

就这点破事，能把 WebSSH、Git、brew 一起干废。

## 先说现场

项目路径在：

```bash
/Users/yy/Documents/Bonjourr
```

传统权限看起来一点问题没有：

```bash
/Users/yy/Documents
drwx------@ yy staff

/Users/yy/Documents/Bonjourr
drwxr-xr-x@ yy staff
```

用户也是 `yy`，目录 owner 也是 `yy`，按普通 Linux/Unix 的直觉，这事应该结束了。

但 WebSSH 里执行：

```bash
git -C /Users/yy/Documents/Bonjourr status --short
```

还是报：

```bash
fatal: Unable to read current working directory: Operation not permitted
```

更恶心的是，报错点甚至不在 `Bonjourr` 这个仓库，而是在当前 shell 正站着的 `~/Documents`。也就是说，`git -C` 还没来得及真正去目标目录，就先死在 `getcwd` 这一步了。

## 一开始能修的都修了

我把问题丢给 Codex，它先按常规文件系统方向排：

```bash
ls -ldeO@ /Users/yy/Documents
ls -ldeO@ /Users/yy/Documents/Bonjourr
xattr -l /Users/yy/Documents/Bonjourr
```

能看到一些 macOS 特有的东西：

```text
com.apple.macl
com.apple.provenance
```

项目目录里没有明显的 `uchg`、`schg`、`restricted`，ACL 也不是那种一眼能看出禁止读取的状态。

然后做了一轮保守修复：

```bash
chmod -RN /Users/yy/Documents/Bonjourr
chflags -R nouchg /Users/yy/Documents/Bonjourr
chown -R yy:staff /Users/yy/Documents/Bonjourr
chmod -R u+rwX /Users/yy/Documents/Bonjourr
xattr -dr com.apple.quarantine /Users/yy/Documents/Bonjourr
```

中间还有个小坑：第一次清 `xattr` 的时候，`.git/objects` 里大量文件报 `Permission denied`。后来发现这些 Git object 是只读文件，恢复用户写权限以后，`xattr` 清理才成功。

这一步修完以后，在普通 shell 里测：

```bash
git -C /Users/yy/Documents/Bonjourr status --short
brew --version
```

都正常。

于是我以为事情结束了。

然后 WebSSH 里一测，还是不行。

## 真正的线索在 code-server 日志里

这时候问题就很明确了：不是文件坏了，是 code-server/WebSSH 这条进程链自己被 macOS 卡住了。

Codex 接着去看 code-server 的启动方式：

```bash
launchctl print gui/501/com.yy.code-server
```

发现 code-server 是一个用户级 LaunchAgent，工作目录是：

```text
/Users/yy
```

这本身没问题。

但再去翻 code-server 的状态和日志，看到它默认打开的是整个：

```text
/Users/yy/Documents
```

日志里已经把话说死了：

```text
EPERM: operation not permitted, scandir '/Users/yy/Documents'
```

这就不是 `chmod` 的事了。这是 macOS TCC 在拦。

## 最后查到 TCC 里有明确拒绝

关键命令是查用户自己的 TCC 数据库：

```bash
sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "select service,client,client_type,auth_value,auth_reason,flags,last_modified
   from access
   where service like '%Documents%'
      or client like '%code-server%'
      or client like '%node%';"
```

结果里面有两条特别关键：

```text
kTCCServiceSystemPolicyDocumentsFolder|/Users/yy/.local/lib/code-server-4.112.0/lib/node|1|2|2|0|...
kTCCServiceSystemPolicyDocumentsFolder|/Users/yy/.local/lib/code-server-4.116.0/lib/node|1|0|2|0|...
```

这里 `auth_value=2` 是允许，`auth_value=0` 是拒绝。

也就是说，旧版 `code-server-4.112.0/lib/node` 能访问 `Documents`，新版 `code-server-4.116.0/lib/node` 被拒绝访问 `Documents`。

这下就全通了。

我只是把 code-server 从 brew 安装换成官方脚本安装，背后的可执行文件路径变了，macOS 就把它当成另一个东西。然后 TCC 里新路径是拒绝状态，于是 WebSSH 里所有依赖当前目录的命令都开始炸。

真他妈离谱。

## 修复方式

先备份 TCC 数据库：

```bash
cp ~/Library/Application\ Support/com.apple.TCC/TCC.db \
   ~/Library/Application\ Support/com.apple.TCC/TCC.db.bak-codeserver-documents-20260521
```

然后只改这一条 code-server 4.116 bundled Node 的权限：

```bash
sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "update access
   set auth_value=2,
       auth_reason=2,
       last_modified=strftime('%s','now')
   where service='kTCCServiceSystemPolicyDocumentsFolder'
     and client='/Users/yy/.local/lib/code-server-4.116.0/lib/node'
     and client_type=1;"
```

再刷新 TCC 缓存并重启 code-server：

```bash
killall tccd
launchctl kickstart -k gui/501/com.yy.code-server
```

另外，Codex 顺手把 code-server 默认打开目录从：

```text
/Users/yy/Documents
```

改成了：

```text
/Users/yy/Documents/Bonjourr
```

这样新开的终端不会再默认站在 `Documents` 根目录上，少踩一个坑。

## 验证

最后用 code-server 同一个 Node 二进制模拟执行：

```bash
/Users/yy/.local/lib/code-server-4.116.0/lib/node -e \
"const cp=require('child_process');
 cp.execFileSync('/usr/bin/git',
 ['-C','/Users/yy/Documents/Bonjourr','status','--short'],
 {cwd:'/Users/yy/Documents',stdio:'inherit'});
 console.log('git-ok')"
```

输出：

```text
git-ok
```

再测 brew：

```bash
/Users/yy/.local/lib/code-server-4.116.0/lib/node -e \
"const cp=require('child_process');
 cp.execFileSync('/opt/homebrew/bin/brew',
 ['--version'],
 {cwd:'/Users/yy/Documents',stdio:'inherit'});
 console.log('brew-ok')"
```

输出：

```text
Homebrew 5.1.7
brew-ok
```

这说明不是我当前这个 shell 碰巧能用，而是 code-server 实际使用的 Node 路径也已经能读 `Documents` 了。

## 最后收一下

这次问题如果只看表象，会很容易被带偏：

```text
brew 报当前目录不可读
git 报 unable to read current working directory
目录 owner/mode 又完全正常
```

但真正的链路是：

```text
code-server 安装方式变化
-> bundled Node 路径变化
-> macOS TCC 把新 Node 当成新客户端
-> Documents 权限是拒绝
-> WebSSH 里当前目录不可读
-> brew/git 全部跟着报错
```

这事最烦的地方就在于，它不是传统权限问题。`chmod`、`chown`、`xattr` 都只能排除干扰，最后真正要看的还是 TCC。

我在旁边能干什么呢？

我只能说：

```text
卧槽，牛逼。
真牛逼。
这都能给你查出来。
```

然后再补一句：code-server 只是从 brew 安装换成官方脚本安装，就能整出这么个屁事，macOS 权限模型有时候是真的让人血压上来。

