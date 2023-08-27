---
title: Lsync+rsync 实现跨机器文件同步
slug: lsync+rsync-for-cross-machine-file-synchronization
tags: [ Linux ]
date: 2022-06-16T21:51:17+08:00
---

Lysncd即Live Syncing Daemon，它是开源的数据实时同步工具(后台进程)，基于inotify和rsync。Lysncd会密切监测本地服务器上的参照目录，当发现目录下有文件或目录变更后，立刻通知远程服务器，并通过rsync 或rsync+ssh方式实现文件同步。这样做的好处就是，你可以利用Lsyncd搭建一个VPS同步镜像，应用场景例如CDN镜像、网站数据备份、网站搬家等等。<!--more-->
发送端需要安装lsyncd，而接收端不需要安装。
lsyncd是基于rsync,故发送端安装lsyncd后会也自动依赖安装rsync。

### **发送端设置SSH免密登录**

如果想要将发送端的数据同步到接收端，发送端必须拥有免密码登录接收端的权限，可以设置密钥登录来完成。

```bash
ssh-keygen -t rsa #生成秘钥直接按回车
ssh-copy-id 目的IP #将公钥发送到目标机器
ssh 192.168.1.101  #测试免密效果
```

### **发送端安装lsync**

```bash
yum -y install epel-release
yum -y install lsyncd
lsyncd --version #查看lsyncd版本
```

### **发送端配置/etc/lsyncd.conf（lua语言通过两个“-”注释）**

```lua
settings {
   logfile = "/var/log/lsyncd.log",  -- 指定日志文件位置
   statusFile = "/var/log/lsyncd.status",  -- 指定状态文件位置
   inotifyMode = "CloseWrite", --当文件被关闭时开始同步
   maxProcesses = 1, --最大线程，ssh模式的缺点就是最大线程只能为1
}
sync {
    default.rsyncssh, --指定工作模式为ssh
    source    = "/home/prometheus-2.31.1.linux-amd64/",  --源路径
    host      = "10.31.140.28",  --目标主机
    targetdir = "/home/prometheus-2.31.1.linux-amd64/", --目标路径
    maxDelays = 1,  --为了实时同步，一般设置为1，表示即使只有一个文件改变也同步
    delay = 0,
    -- init = false,
    rsync    = {
        binary = "/usr/bin/rsync",--告诉lsync,rsync的地址在哪里
        archive = true, -- 打包后再同步(打包不等于压缩，打包即可以压缩也可以不压缩)
        compress = true,-- 压缩后再同步
        verbose  = false,-- 输出同步信息 
       _extra = {
      "--include=rules.yml",--同步的文件
      "--exclude=*",--排除掉的文件
          },
        },
        ssh   = {
        port  =  22
        }
    }
```

### **发送端启动lsyncd**

```bash
systemctl start lsyncd
systemctl enabled lsyncd #配置开机启动
```

### **接收端安装rsync**

检查是否已安装，rsync配置文件不用做修改。

```bash
rpm -qa |grep rsync #rsync是git的依赖，所以如果装了git，就会有rsync
netstat -an | grep 873 #检查rsync服务是否启动
systemctl start rsyncd
systemctl status rsyncd #查看服务状态
systemctl enable rsyncd.service
kill $(ps -aux |grep rsync |awk '{print $2}') # 关闭lsync
```

### **设置lsync开启自启动**

写进/etc/rc.local

```bash
/usr/bin/lsyncd /etc/lsyncd.conf #启动lsync
#kill $(ps -aux |grep lsync |awk '{print $2}') # 关闭lsync
```

### **测试成功**
