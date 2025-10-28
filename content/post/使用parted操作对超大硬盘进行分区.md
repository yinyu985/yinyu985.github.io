---
title: 使用 parted 操作对超大硬盘进行分区
slug: partitioning-a-large-hard-drive-using-the-parted-operation
tags: [ Linux ]
date: 2022-12-25T17:46:13+08:00
---

我承认我可能是标题党了，因为每个人对“大”的理解肯定是不一样的，更何况我说的是超大。其实就是 10TB，对我一个没见过大世面的行业新人和 fdisk 来说，它确实算大的了，本文介绍对 parted 的学习和实际使用。<!--more-->

`parted` 是由 GNU 组织开发的一款功能强大的磁盘分区和分区大小调整工具，与 fdisk 不同，它支持调整分区的大小。作为一种设计用于 Linux 的工具，它没有构建成处理与 fdisk 关联的多种分区类型，但是，它可以处理最常见的分区格式，包括：ext2、ext3、fat16、fat32、NTFS、ReiserFS、JFS、XFS、UFS、HFS 以及 Linux 交换分区。

## 前情提要

> 1. 我们可以使用 `fdisk` 命令对硬盘进行快速的分区，但对高于 2TB 的硬盘分区，此命令却无能为力。
> 2. MBR 分区表仅支持最大四个主分区，且不支持 2TB 以上的磁盘，因此，大磁盘更适合使用 parted 进行 GPT 的分区。
> 3. fdisk 工具虽然很好用，但对于大于 2T 以上的硬盘分区特别慢，可能一部分容量识别不了，也不支持非交互模式。

```bash
### 查看现在的硬盘容量
[root@localhost ~]# df -mh
文件系统                 容量  已用  可用 已用% 挂载点
devtmpfs                 7.8G     0  7.8G    0% /dev
tmpfs                    7.8G     0  7.8G    0% /dev/shm
tmpfs                    7.8G  124M  7.7G    2% /run
tmpfs                    7.8G     0  7.8G    0% /sys/fs/cgroup
/dev/mapper/centos-root  2.3T  2.2T  181G   93% /
/dev/sda1                197M  128M   69M   66% /boot
tmpfs                    1.6G     0  1.6G    0% /run/user/0

### 查看现在一共有多少块盘，可以看到 sdd 是 10T，可惜不是 SSD 哈哈哈。
[root@localhost ~]# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0  300G  0 disk 
├─sda1            8:1    0  200M  0 part /boot
├─sda2            8:2    0 59.8G  0 part 
│ ├─centos-root 253:0    0  2.3T  0 lvm  /
│ └─centos-swap 253:1    0    8G  0 lvm  [SWAP]
└─sda3            8:3    0  240G  0 part 
  └─centos-root 253:0    0  2.3T  0 lvm  /
sdb               8:16   0    1T  0 disk 
└─sdb1            8:17   0 1024G  0 part 
  └─centos-root 253:0    0  2.3T  0 lvm  /
sdc               8:32   0    1T  0 disk 
└─sdc1            8:33   0 1024G  0 part 
  └─centos-root 253:0    0  2.3T  0 lvm  /
sdd               8:48   0   10T  0 disk 
sr0              11:0    1  973M  0 rom  

### fdisk 头铁试一试，其实是我一开始并不知道 fdisk 对于大硬盘不好使。
[root@localhost ~]# fdisk -l

磁盘 /dev/sda：322.1 GB, 322122547200 字节，629145600 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x000d93fa

   设备 Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048      411647      204800   83  Linux
/dev/sda2          411648   125829119    62708736   8e  Linux LVM
/dev/sda3       125829120   629145599   251658240   83  Linux

磁盘 /dev/mapper/centos-root：2512.3 GB, 2512329375744 字节，4906893312 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节

磁盘 /dev/mapper/centos-swap：8589 MB, 8589934592 字节，16777216 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节

磁盘 /dev/sdb：1099.5 GB, 1099511627776 字节，2147483648 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x3d4c36cf

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048  2147483647  1073740800   83  Linux

磁盘 /dev/sdc：1099.5 GB, 1099511627776 字节，2147483648 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x4125e7ff

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdc1            2048  2147483647  1073740800   83  Linux

磁盘 /dev/sdd：10995.1 GB, 10995116277760 字节，21474836480 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节

### 还没有意识到问题的严重性
[root@localhost ~]# fdisk /dev/sdd 
欢迎使用 fdisk (util-linux 2.23.2)。

更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。

Device does not contain a recognized partition table
使用磁盘标识符 0x4dde7e22 创建新的 DOS 磁盘标签。

WARNING: The size of this disk is 11.0 TB (10995116277760 bytes).
DOS partition table format can not be used on drives for volumes
larger than (2199023255040 bytes) for 512-byte sectors. Use parted(1) and GUID 
partition table format (GPT).
### 上面直接写了 larger than，叫你用 parted 并且叫你用 GPT 格式，然而我没看到

命令(输入 m 获取帮助)：n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): 
Using default response p
分区号 (1-4，默认 1)：
起始 扇区 (2048-4294967295，默认为 2048)：
将使用默认值 2048
Last 扇区, +扇区 or +size{K,M,G} (2048-4294967294，默认为 4294967294)：
将使用默认值 4294967294
分区 1 已设置为 Linux 类型，大小设为 2 TiB

命令(输入 m 获取帮助)：n
Partition type:
   p   primary (1 primary, 0 extended, 3 free)
   e   extended
Select (default p): 
Using default response p
分区号 (2-4，默认 2)：
起始 扇区 (4294967295-4294967295，默认为 4294967295)：
将使用默认值 4294967295
分区 2 已设置为 Linux 类型，大小设为 512 B

命令(输入 m 获取帮助)：p

磁盘 /dev/sdd：10995.1 GB, 10995116277760 字节，21474836480 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x4dde7e22

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdd1            2048  4294967294  2147482623+  83  Linux
/dev/sdd2      4294967295  4294967295           0+  83  Linux
### 新建一个发现只有一个好使，第二个是 0，灰溜溜的不保存退出了
命令(输入 m 获取帮助)：q

### 关于保存退出，需要注意的是 parted 中所有的操作都是立即生效的，没有保存生效的概念。这一点和 fdisk 交互命令明显不同，所以做的所有操作大家要加倍小心。
[root@localhost ~]# parted /dev/sdd
GNU Parted 3.1
使用 /dev/sdd
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) p                                                                
错误: /dev/sdd: unrecognised disk label

### 这里报错不认识的硬盘标签，没事，后面就认识了。
Model: VMware Virtual disk (scsi)                                         
Disk /dev/sdd: 11.0TB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags: 

(parted) mklabel gpt
### 修改分区表命令

(parted) mkpart
### 输入创建分区命令，后面不要参数，全部靠交互
分区名称？  []? sdd1
文件系统类型？  [ext2]? xfs                                      
起始点？ 0                                                                
结束点？ 11.0TB                                                           
警告: The resulting partition is not properly aligned for best performance.
### 这里需要注意了，它说这样分的话，没有正确对齐的分区会影响硬盘的性能。
忽略/Ignore/放弃/Cancel? cancel    

### 于是我取消了这一步操作，这个跟前面的注意事项不冲突，毕竟这是他跟我说可以取消的。
(parted) q                                                                
信息: You may need to update /etc/fstab.

[root@localhost ~]# lsblk    
### 一看硬盘没啥变化。
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0  300G  0 disk 
├─sda1            8:1    0  200M  0 part /boot
├─sda2            8:2    0 59.8G  0 part 
│ ├─centos-root 253:0    0  2.3T  0 lvm  /
│ └─centos-swap 253:1    0    8G  0 lvm  [SWAP]
└─sda3            8:3    0  240G  0 part 
  └─centos-root 253:0    0  2.3T  0 lvm  /
sdb               8:16   0    1T  0 disk 
└─sdb1            8:17   0 1024G  0 part 
  └─centos-root 253:0    0  2.3T  0 lvm  /
sdc               8:32   0    1T  0 disk 
└─sdc1            8:33   0 1024G  0 part 
  └─centos-root 253:0    0  2.3T  0 lvm  /
sdd               8:48   0   10T  0 disk 
sr0              11:0    1  973M  0 rom

[root@localhost ~]# fdisk -l  
### 再看看，发现还是有变化的，至少变成了 GPT。

磁盘 /dev/sda：322.1 GB, 322122547200 字节，629145600 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x000d93fa

   设备 Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048      411647      204800   83  Linux
/dev/sda2          411648   125829119    62708736   8e  Linux LVM
/dev/sda3       125829120   629145599   251658240   83  Linux

磁盘 /dev/mapper/centos-root：2512.3 GB, 2512329375744 字节，4906893312 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节

磁盘 /dev/mapper/centos-swap：8589 MB, 8589934592 字节，16777216 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节

磁盘 /dev/sdb：1099.5 GB, 1099511627776 字节，2147483648 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x3d4c36cf

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048  2147483647  1073740800   83  Linux

磁盘 /dev/sdc：1099.5 GB, 1099511627776 字节，2147483648 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x4125e7ff

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdc1            2048  2147483647  1073740800   83  Linux

WARNING: fdisk GPT support is currently new, and therefore in an experimental phase. Use at your own discretion.

磁盘 /dev/sdd：10995.1 GB, 10995116277760 字节，21474836480 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：gpt
Disk identifier: FE3E929B-4D5C-440D-8373-92F8AB438821

Start          End    Size  Type            Name

### 查了一遍资料的我又回来了，刚才说没对齐就是因为下面的四个参数，正确的做法还需要计算，不过好在这是一块新硬盘，没数据，搞不坏。
[root@localhost ~]# cat /sys/block/sdd/queue/optimal_io_size
0
[root@localhost ~]# cat /sys/block/sdd/queue/minimum_io_size
512
[root@localhost ~]# cat /sys/block/sdd/alignment_offset
0
[root@localhost ~]# cat /sys/block/sdd/queue/physical_block_size
512

[root@localhost ~]# parted /dev/sdd
GNU Parted 3.1
使用 /dev/sdd
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) p                                                                
Model: VMware Virtual disk (scsi)
Disk /dev/sdd: 11.0TB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start  End  Size  File system  Name  标志

(parted) print
Model: VMware Virtual disk (scsi)
Disk /dev/sdd: 11.0TB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start  End  Size  File system  Name  标志

(parted) mkpart primary 0.00T 100%
### 这样就不用计算了，也不会存在没对齐的情况，parted /dev/sdb mkpart primary xfs 0% 100%，这是一条非交互式命令，在 mkpart 的时候选中整个硬盘，新建为一个分区，和我所使用的命令类似。至于如何创建多个分区，请查看 mkpart 的具体用法。

(parted) p                                                                
Model: VMware Virtual disk (scsi)
Disk /dev/sdd: 11.0TB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name     标志
 1      1049kB  11.0TB  11.0TB               primary

(parted) align-check optimal 1                                            
1 aligned

(parted) quit                                                             
信息: You may need to update /etc/fstab.
### 这里提示你可能需要更新 /etc/fstab

[root@localhost ~]# lsblk
### 10T 硬盘，一个分区，其实稳妥起见多分几个区是最好的，不至于一坏全坏，但是谁叫咱这是小作坊呢。爱咋咋滴！
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0  300G  0 disk 
├─sda1            8:1    0  200M  0 part /boot
├─sda2            8:2    0 59.8G  0 part 
│ ├─centos-root 253:0    0  2.3T  0 lvm  /
│ └─centos-swap 253:1    0    8G  0 lvm  [SWAP]
└─sda3            8:3    0  240G  0 part 
  └─centos-root 253:0    0  2.3T  0 lvm  /
sdb               8:16   0    1T  0 disk 
└─sdb1            8:17   0 1024G  0 part 
  └─centos-root 253:0    0  2.3T  0 lvm  /
sdc               8:32   0    1T  0 disk 
└─sdc1            8:33   0 1024G  0 part 
  └─centos-root 253:0    0  2.3T  0 lvm  /
sdd               8:48   0   10T  0 disk 
└─sdd1            8:49   0   10T  0 part 
sr0              11:0    1  973M  0 rom

[root@localhost ~]# cat /etc/fstab 
#/etc/fstab Created by anaconda on Sat Oct 22 16:59:09 2022
#Accessible filesystems, by reference, are maintained under '/dev/disk'
See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=74925622-e002-42b3-aa1c-034b87432c32 /boot                   xfs     defaults        0 0
/dev/mapper/centos-swap swap                    swap    defaults        0 0

[root@localhost ~]# partprobe
### 后面就是新建 PV 加到 VG 的 LVM 扩容操作了
[root@localhost ~]# pvscan
  PV /dev/sda2   VG centos          lvm2 [59.80 GiB / 0    free]
  PV /dev/sda3   VG centos          lvm2 [<240.00 GiB / 0    free]
  PV /dev/sdb1   VG centos          lvm2 [<1024.00 GiB / 0    free]
  PV /dev/sdc1   VG centos          lvm2 [<1024.00 GiB / 0    free]
  Total: 4 [2.29 TiB] / in use: 4 [2.29 TiB] / in no VG: 0 [0   ]

[root@localhost ~]# pvcreate /dev/sdd1
  Physical volume "/dev/sdd1" successfully created.

[root@localhost ~]# pvs
  PV         VG     Fmt  Attr PSize     PFree  
  /dev/sda2  centos lvm2 a--     59.80g      0 
  /dev/sda3  centos lvm2 a--   <240.00g      0 
  /dev/sdb1  centos lvm2 a--  <1024.00g      0 
  /dev/sdc1  centos lvm2 a--  <1024.00g      0 
  /dev/sdd1         lvm2 ---    <10.00t <10.00t

[root@localhost ~]# lvs
  LV   VG     Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root centos -wi-ao---- 2.28t                                                    
  swap centos -wi-ao---- 8.00g                                                    

[root@localhost ~]# vgextend centos /dev/sdd1
  Volume group "centos" successfully extended

[root@localhost ~]# lvextend -l +100%FREE /dev/centos/root
  Size of logical volume centos/root changed from 2.28 TiB (598986 extents) to 12.28 TiB (3220425 extents).
  Logical volume centos/root successfully resized.

[root@localhost ~]# df -T
文件系统                类型          1K-块       已用      可用 已用% 挂载点
devtmpfs                devtmpfs    8111856          0   8111856    0% /dev
tmpfs                   tmpfs       8123796          0   8123796    0% /dev/shm
tmpfs                   tmpfs       8123796     126632   7997164    2% /run
tmpfs                   tmpfs       8123796          0   8123796    0% /sys/fs/cgroup
/dev/mapper/centos-root xfs      2453420136 2278811760 174608376   93% /
/dev/sda1               xfs          201380     130924     70456   66% /boot
tmpfs                   tmpfs       1624760          0   1624760    0% /run/user/0

[root@localhost ~]# xfs_growfs /dev/centos/data
xfs_growfs: /dev/centos/data is not a mounted XFS filesystem
### 这里有一处小细节，格式化以前都是这么干，没出问题，这次报错了，查了一下，因为它太大了，只能格式化挂载点

[root@localhost ~]# df -T
文件系统                类型          1K-块       已用      可用 已用% 挂载点
devtmpfs                devtmpfs    8111856          0   8111856    0% /dev
tmpfs                   tmpfs       8123796          0   8123796    0% /dev/shm
tmpfs                   tmpfs       8123796     126632   7997164    2% /run
tmpfs                   tmpfs       8123796          0   8123796    0% /sys/fs/cgroup
/dev/mapper/centos-root xfs      2453420136 2280066416 173353720   93% /
/dev/sda1               xfs          201380     130924     70456   66% /boot
tmpfs                   tmpfs       1624760          0   1624760    0% /run/user/0

[root@localhost ~]# xfs_growfs /
meta-data=/dev/mapper/centos-root isize=512    agcount=181, agsize=3394816 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=613361664, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=6630, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 613361664 to 3297715200
### 扩容成功，圆满完成。

[root@localhost ~]# df -T
文件系统                类型           1K-块       已用        可用 已用% 挂载点
devtmpfs                devtmpfs     8111856          0     8111856    0% /dev
tmpfs                   tmpfs        8123796          0     8123796    0% /dev/shm
tmpfs                   tmpfs        8123796     126632     7997164    2% /run
tmpfs                   tmpfs        8123796          0     8123796    0% /sys/fs/cgroup
/dev/mapper/centos-root xfs      13190834280 2278001048 10912833232   18% /
/dev/sda1               xfs           201380     130924       70456   66% /boot
tmpfs                   tmpfs        1624760          0     1624760    0% /run/user/0

[root@localhost ~]# df -Th
文件系统                类型      容量  已用  可用 已用% 挂载点
devtmpfs                devtmpfs  7.8G     0  7.8G    0% /dev
tmpfs                   tmpfs     7.8G     0  7.8G    0% /dev/shm
tmpfs                   tmpfs     7.8G  124M  7.7G    2% /run
tmpfs                   tmpfs     7.8G     0  7.8G    0% /sys/fs/cgroup
/dev/mapper/centos-root xfs        13T  2.2T   11T   18% /
/dev/sda1               xfs       197M  128M   69M   66% /boot
tmpfs                   tmpfs     1.6G     0  1.6G    0% /run/user/0
```

## 参考资料

[parted 命令用法详解：创建分区](http://c.biancheng.net/view/905.html)

[parted：磁盘分区和分区大小调整工具](https://wangchujiang.com/linux-command/c/parted.html)

[parted 的详解及常用分区使用方法](https://www.cnblogs.com/lvzhenjiang/p/14391479.html)