---
title: SSH 免密配置及 Ansible 基础知识
slug: ssh-secret-free-configuration-and-ansible-basics
tags: [ Linux, Ansible ]
date: 2021-10-12T00:14:51+08:00
---

**Secure Shell**（安全外壳协议，简称 **SSH**）是一种加密的[网络传输协议](https://zh.wikipedia.org/wiki/网络传输协议)，在不安全的网络中为网络服务提供安全的传输。

**Ansible** 是 Python 中的一套模块，一套自动化工具，只需要使用 SSH 协议连接即可用来系统管理、自动化执行命令等任务。

<!--more-->

## 如何配置免密登录

免密登录，只需三步。

- 首先在 SSH 服务端配置允许公钥私钥配对认证；
- 然后在客户端生成公钥；
- 最后将客户端的公钥上传到服务端。

这样就可以从客户端免密登录服务端特定的用户了。

## 在服务端上（提供 SSH 服务，就是你登录的那台机器）进行如下操作

以 root 用户登录，更改 SSH 配置文件 `/etc/ssh/sshd_config`，去除以下配置的注释，然后重启 `sshd` 服务：

```bash
RSAAuthentication yes                    # 启用 RSA 认证
PubkeyAuthentication yes               # 启用公钥私钥配对认证方式
AuthorizedKeysFile .ssh/authorized_keys # 公钥文件路径
systemctl restart sshd                   # 重启 SSH 服务
```

## 在客户端上执行

```bash
[root@client /]# ssh-keygen -t rsa
```

一路默认回车，系统在当前目录下，也就是 `/root/.ssh` 下生成 `id_rsa`、`id_rsa.pub`。

## 把 id_rsa.pub 发送到服务端机器上

```bash
[root@client /]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.11.20
```

这条命令是将刚才生成的公钥发送到你要登录的服务端保存。

## 验证

```bash
ssh root@192.168.11.20
```

如果当前用户和你要登录到的机器的用户都是 root，那么你可以直接：

```bash
ssh 192.168.11.20
```

## Ansible 基础知识

## 1. 什么是 Ansible

Ansible 是 Python 中的一套模块，系统中的一套自动化工具，只需要使用 SSH 协议连接即可用来系统管理、自动化执行命令等任务。

## 2. Ansible 优势

1. Ansible 不需要单独安装客户端，也不需要启动任何服务；
2. Ansible 是 Python 中的一套完整的自动化执行任务模块；
3. Ansible Playbook 采用 YAML 配置，对于自动化任务执行一目了然；
4. Ansible 模块较多，对于自动化的场景支持较丰富。

![image.png](https://s2.loli.net/2022/12/26/T23mnbhptA1sNWK.png)

**Ansible Playbook**

任务剧本（又称任务集），编排定义 Ansible 任务集的配置文件，由 Ansible 顺序依次执行，YAML 格式。

**Inventory**

Ansible 管理主机的清单，默认是 `/etc/ansible/hosts` 文件。

**Modules**

Ansible 执行命令的功能模块，Ansible 2.3 版本为止，共有 1039 个模块。还可以自定义模块。

**Plugins**

插件，模块功能的补充，常有连接类型插件、循环插件、变量插件、过滤插件，插件功能用的较少。  
`/etc/ansible/hosts` 文件，用于定义被管理主机的认证信息，例如 SSH 登录用户名、密码以及 key 相关信息。

1. 主机支持主机名通配以及正则表达式，例如 `web[1:3].test.com` 代表三台主机；
2. 主机支持基于非标准的 SSH 端口，例如 `web1.test.com:6666`；
3. 主机支持指定变量，可对个别主机的特殊配置，如登录用户、密码。

![image.png](https://s2.loli.net/2022/12/26/pErH842CNLqhiGu.png)

Ansible 执行返回 -> 颜色信息说明：

- 黄色：对远程节点进行相应修改；
- 绿色：对远程节点不进行相应修改，或者只是对远程节点信息进行查看；
- 红色：操作执行命令有异常；
- 紫色：表示对命令执行发出警告信息（可能存在的问题，给你一下建议）。

## script 脚本模块

```bash
ansible test -m script -a "/server/scripts/yum.sh"
# 在本地运行模块，等同于在远程执行，不需要将脚本文件进行推送到目标主机执行
```

## yum 安装软件模块

```bash
ansible test -m yum -a 'name=httpd state=installed'
```

## copy 文件拷贝模块

```bash
ansible test -m copy -a "src=/etc/hosts dest=/tmp/test.txt"
ansible test -m copy -a "src=/etc/hosts dest=/tmp/test.txt backup=yes"
# 在推送覆盖远程端文件前，对远端已有文件进行备份，按照时间信息备份
ansible test -m copy -a "content='bgx' dest=/tmp/test"
# 直接向远端文件内写入数据信息，并且会覆盖远端文件内原有数据信息

# 参数说明：
src     # 推送数据的源文件信息
dest    # 推送数据的目标路径
backup  # 对推送传输过去的文件，进行备份
content # 直接批量在被管理端文件中添加内容
group   # 将本地文件推送到远端，指定文件属组信息
owner   # 将本地文件推送到远端，指定文件属主信息
mode    # 将本地文件推送到远端，指定文件权限信息
```

## service 服务模块

```bash
[root@m01 ~]# ansible test -m service -a "name=crond state=stopped enabled=yes"

# 参数说明：
name        # 定义要启动服务的名称
state       # 指定服务状态
    started     # 启动服务
    stopped     # 停止服务
    restarted   # 重启服务
    reloaded    # 重载服务
enabled         # 开机自启
```

资料：<https://halysl.github.io/2020/03/11/ansible%E5%9F%BA%E6%9C%AC%E6%93%8D%E4%BD%9C%E5%8F%8A%E5%BA%94%E7%94%A8/#%E6%A0%B8%E5%BF%83%E6%A8%A1%E5%9D%97-playbook>

## raw 模块

raw 模块的功能与 shell 和 command 类似。但 raw 模块运行时不需要在远程主机上配置 Python 环境。

## command 模块

command 模块用于运行系统命令。不支持管道符和变量等（`<`, `>`, `|`, 和 `&` 等），如果要使用这些，那么可以使用 shell 模块。在使用 Ansible 的时候，默认的模块是 `-m command`，因此模块的参数不需要填写，直接使用即可。

## shell 模块

shell 模块负责在被 Ansible 控制的节点（服务器）执行命令行。shell 模块是通过 `/bin/sh` 进行执行，所以 shell 模块可以执行任何命令，就像在本机执行一样。

## script 模块

script 模块将控制节点的脚本执行在被控节点上。