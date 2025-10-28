---
title: LVM 创建和扩容
slug: lvm-creation-and-expansion
tags: [Linux]
date: 2022-08-14T22:14:36+08:00
---

LVM 是 Logical Volume Manager（逻辑卷管理器）的简写，它是 Linux 环境下对磁盘分区进行管理的一种机制。LVM 将一个或多个磁盘分区（PV）虚拟为一个卷组（VG），相当于一个大的硬盘，我们可以在上面划分一些逻辑卷（LV）。当卷组的空间不够使用时，可以将新的磁盘分区加入进来。我们还可以从卷组剩余空间上划分一些空间给空间不够用的逻辑卷使用。<!--more-->

![](https://s2.loli.net/2022/08/20/HIeMkpU5TbfzmtN.png)

# 创建一个 LVM

将硬盘或者硬盘分组，创建为物理卷（PV）>>> 将 PV 加入到卷组（VG）>>> 将卷组分成逻辑卷（LV）

首先确定系统中是否安装了 lvm 工具：

```bash
rpm -qa | grep lvm
```

如果命令结果输出 rpm 版本，那么说明系统已经安装了 LVM 管理工具；如果命令没有输出则说明没有安装 LVM 管理工具，则需要从网络下载或者从光盘安装 LVM rpm 工具包。

```bash
yum install -y lvm2
```

## 查看硬盘

```bash
lsblk # 也可以通过 fdisk -l 查看硬盘
```

![](https://s2.loli.net/2022/08/20/nJB3aSkOzVWN4YA.png)

## 对硬盘进行分区

fdisk 里面的具体操作，可输入 `m` 进行帮助。最常用的是 `n`（新建）、`d`（删除）、`p`（打印）、`q`（退出）、`t`（修改系统标识符）、`w`（写入并退出）。

> 也可以不分区，我这里就没有分区，分区的意义在于，不把鸡蛋放在同一个篮子里，格式化时不用全部格式化，可以保留部分文件。

改变系统标识符：  
输入 `t` 改变分区 1 的属性  
输入 `L` 查看属性对应的命令  
输入 `8e` 改变分区 1 为 Linux LVM 格式  
输入 `p` 打印分区情况查看是否是 Linux LVM 格式

> 没有修改分区的属性，添加到 lvm 里面也没有出问题，不知为何，可能 `pvcreate` 时已经完成了相应操作。

## 创建物理卷

```bash
pvcreate /dev/vdb /dev/vdc /dev/vdd /dev/vde /dev/vdf
```

![](https://s2.loli.net/2022/08/20/tk4m9aEjLn5zV7d.png)

## 创建卷组

创建一个卷组把刚才创建的物理卷加入到卷组：

```bash
vgcreate lvm_01 /dev/vdb /dev/vdc /dev/vdd /dev/vde /dev/vdf
```

![](https://s2.loli.net/2022/08/20/H2q9A4n1CNlwdQs.png)

## 创建逻辑卷

```bash
lvcreate -L 150g -n lv01 lvm_01
```

用 `lvm_01` 这个卷组创建一个逻辑卷名叫 `lv01`，`-L` 用于指定逻辑卷的大小，支持单位 M、G、T。

![](https://s2.loli.net/2022/08/20/qNjA9c1EsZkbyRL.png)

使用以下命令查看逻辑卷详细信息：

```bash
lvdisplay
```

## 给现有的 LVM 扩容

![](https://s2.loli.net/2022/08/20/nm8iRBaYfkT3jlZ.png)

此时，LVM 已经创建完成，需要创建文件系统后挂载。

## 创建文件系统并挂载

创建一个 ext4 文件系统：

```bash
mkfs.ext4 /dev/lvm_01/lv01
```

创建一个 xfs 文件系统：

```bash
mkfs.xfs /dev/lvm_01/lv01
```

创建挂载点，例如 `/media/lv01`。  
如果已有的挂载点，可以跳过此步骤。

```bash
mkdir /media/lv01
```

将创建的文件系统挂载到挂载点：

```bash
mount /dev/lvm_01/lv01 /media/lv01
```

# 给现有逻辑卷扩容

`fdisk -l` 查看有一块新加的硬盘 `/dev/sdd`：

```bash
fdisk -l
磁盘 /dev/sda：32.2 GB, 32212254720 字节，62914560 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 4096 字节
I/O 大小(最小/最佳)：4096 字节 / 4096 字节
磁盘标签类型：dos
磁盘标识符：0x000ecbd2
   设备 Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     1026047      512000   83  Linux
/dev/sda2         1026048    62914559    30944256   83  Linux
磁盘 /dev/sdb：1099.5 GB, 1099511627776 字节，2147483648 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 4096 字节
I/O 大小(最小/最佳)：4096 字节 / 4096 字节
磁盘标签类型：dos
磁盘标识符：0x05008ad2
   设备 Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048  2147483647  1073740800   83  Linux
磁盘 /dev/mapper/centos-data：2194.7 GB, 2194724093952 字节，4286570496 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 4096 字节
I/O 大小(最小/最佳)：4096 字节 / 4096 字节
磁盘 /dev/sdc：1099.5 GB, 1099511627776 字节，2147483648 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 4096 字节
I/O 大小(最小/最佳)：4096 字节 / 4096 字节
磁盘标签类型：dos
磁盘标识符：0x10a5ae44
   设备 Boot      Start         End      Blocks   Id  System
/dev/sdc1            2048  2147483647  1073740800   83  Linux
磁盘 /dev/sdd：1099.5 GB, 1099511627776 字节，2147483648 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 4096 字节
I/O 大小(最小/最佳)：4096 字节 / 4096 字节
```

## 对磁盘进行分区

```bash
fdisk /dev/sdd
```

## 使内核重新读取分区表

```bash
partprobe
```

## 创建物理卷

扫描系统 PV：

```bash
pvscan
```

创建 PV：

```bash
pvcreate /dev/sdd1
```

查看 PV：

```bash
pvdisplay
```

## 查看现有的卷组

```bash
lvdisplay
  --- Logical volume ---
  LV Path                /dev/centos/data
  LV Name                data
  VG Name                centos
  LV UUID                1annzJ-OeMD-GGAi-Ej1N-mkSV-jnNB-4IFj0d
  LV Write Access        read/write
  LV Creation host, time ElasticSearch-node3, 2022-03-03 16:07:24 +0800
  LV Status              available
  # open                 1
  LV Size                <3.00 TiB
  Current LE             786429
  Segments               4
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0
```

## 将新建的物理卷加到卷组

```bash
vgextend centos /dev/sdd1
```

## 将卷组里空余空间加到逻辑卷

```bash
lvextend -l +100%FREE /dev/centos/data
```

加完了并没有万事大吉。  
查看文件系统的类型：

```bash
df -T
```

```bash
文件系统                类型        1K-块    已用     可用 已用% 挂载点
/dev/mapper/centos-root xfs      52403200 1913936 50489264    4% /
devtmpfs                devtmpfs  1964224       0  1964224    0% /dev
tmpfs                   tmpfs     1976320       0  1976320    0% /dev/shm
tmpfs                   tmpfs     1976320   11908  1964412    1% /run
tmpfs                   tmpfs     1976320       0  1976320    0% /sys/fs/cgroup
/dev/sda1               xfs       1038336  135236   903100   14% /boot
/dev/mapper/centos-home xfs      26324420   32992 26291428    1% /home
tmpfs                   tmpfs      395268       0   395268    0% /run/user/0
```

```bash
xfs_growfs /dev/centos/data   ## 当逻辑卷文件系统为 xfs 使用
resize2fs /dev/centos/data    ## 当逻辑卷文件系统为 ext4 使用
```