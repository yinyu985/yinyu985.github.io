---
title: MySQL 的 MHA 高可用安装踩坑记录
slug: mysql's-mha-high-availability-installation-crater-record
tags: [ Linux, MySQL ]
date: 2022-10-18T21:44:03+08:00
---

MHA是一种MySQL高可用解决方案，可用于Position或者GTID模式下的主从复制架构，可以在主从故障时自动完成主从切换，并且最大程度的去保持数据一致性。MHA由管理节点（Manager）和数据节点（Node）组成，一套MHA Manager可以管理多套MySQL集群。当Manager发现MySQL Master出现故障时自动将一个拥有最新数据的Slave提升为Master，并让另外的Slave重新指向到新的Master上来。

<!--more-->

> 在MySQL故障切换过程中，MHA能做到在0~30秒之内自动完成数据库的故障切换操作，并且在进行故障切换的过程中，MHA能在最大程度上保证数据的一致性，以达到真正意义上的高可用。
>
> **该软件由两部分组成：MHA Manager（管理节点）和MHA Node（数据节点）**。MHA Manager可以单独部署在一台独立的机器上管理多个master-slave集群，也可以部署在一台slave节点上。MHA Node运行在每台MySQL服务器上，MHA Manager会定时探测集群中的master节点，当master出现故障时，它可以自动将最新数据的slave提升为新的master，然后将所有其他的slave重新指向新的master。整个故障转移过程对应用程序完全透明。
>
> 在MHA自动故障切换过程中，MHA试图从宕机的主服务器上保存二进制日志，最大程度的保证数据的不丢失，但这并不总是可行的。例如，如果主服务器硬件故障或无法通过ssh访问，MHA没法保存二进制日志，只进行故障转移而丢失了最新的数据。使用MySQL 5.5的半同步复制，可以大大降低数据丢失的风险。MHA可以与半同步复制结合起来。如果只有一个slave已经收到了最新的二进制日志，MHA可以将最新的二进制日志应用于其他所有的slave服务器上，因此可以保证所有节点的数据一致性。
>
> 目前MHA主要支持一主多从的架构，要搭建MHA,要求一个复制集群中必须最少有三台数据库服务器，一主二从，即一台充当master，一台充当备用master，另外一台充当从库，因为至少需要三台服务器

### 准备工作

#### 编写hosts、hostname

让集群里的每台机器都互相认识（如无特殊说明，以下操作均要在三台机器执行）

```bash
三台机器分别设置hostname，然后添加hosts。
hostnamectl set-hostname mysql_master
hostnamectl set-hostname mysql_slave1
hostnamectl set-hostname mysql_slave2
cat >> /etc/hosts << EOF
10.23.188.107  mysql_master
10.23.188.91   mysql_slave1
10.23.188.92   mysql_slave2
EOF
```

#### 配置免密登录

生成公钥，让三台机器可以互相免密登录，这个特性也让MHA变得看起来不是特别安全。

```bash
ssh-keygen -t rsa
一路回车
for i in mysql_master mysql_slave1 mysql_slave2;do ssh-copy-id $i;done
for循环发送到三台机器
```

#### 安装MySQL

CentOS 7将MySQL从默认的yum源中移除，用mariadb代替了,所以单独导入安装源 

```bash
查看系统自带的Mariadb
rpm -qa|grep -i mariadb
卸载系统自带的Mariadb
rpm -qa|grep -i mariadb |xargs rpm -e
导入安装源
rpm -ivh https://repo.mysql.com/mysql57-community-release-el7-9.noarch.rpm
yum install mysql-community-server
报错
"MySQL 5.7 Community Server" 的 GPG 密钥已安装，但是不适用于此软件包。请检查源的公钥 URL 是否配置正确。
失败的软件包是：mysql-community-libs-compat-5.7.37-1.el7.x86_64
yum install mysql-community-server --nogpgcheck
加入--nogpgcheck配置跳过校验。
```

#### MySQL配置

```bash
touch /etc/my.cnf
chmod 644 /etc/my.cnf
cat /etc/my.cnf
[client]
port = 3307
socket = /tmp/mysql.sock
#character-set-server = utf8
[mysqld]
port = 3307
socket = /var/run/mysqld/mysqld.sock
pid-file = /var/run/mysqld/mysqld.pid
user = mysql
bind-address = 0.0.0.0
server-id = 1 
skip-external-locking 
explicit_defaults_for_timestamp
datadir = /home/mysql_data
init-connect = 'SET NAMES utf8'
#character-set-server = utf8
#skip-name-resolve
back_log = 300
max_connections = 1000
max_connect_errors = 6000
open_files_limit = 65535
table_open_cache = 128
max_allowed_packet = 4M
binlog_cache_size = 1M
max_heap_table_size = 8M
tmp_table_size = 16M
read_buffer_size = 2M
read_rnd_buffer_size = 8M
sort_buffer_size = 8M
join_buffer_size = 8M
#key_buffer_size = 4M
thread_cache_size = 8
query_cache_type = 1
query_cache_size = 8M
query_cache_limit = 2M
ft_min_word_len = 4
performance_schema = 0
explicit_defaults_for_timestamp
lower_case_table_names = 1
skip-external-locking
default_storage_engine = InnoDB
innodb_file_per_table = 1
innodb_open_files = 500
innodb_buffer_pool_size = 128M
innodb_write_io_threads = 4
innodb_read_io_threads = 4
innodb_thread_concurrency = 0
innodb_purge_threads = 1
innodb_flush_log_at_trx_commit = 2
innodb_log_buffer_size = 2M
innodb_log_file_size = 32M
innodb_log_files_in_group = 3
innodb_max_dirty_pages_pct = 90
innodb_lock_wait_timeout = 120
bulk_insert_buffer_size = 8M
myisam_sort_buffer_size = 8M
myisam_max_sort_file_size = 10G
interactive_timeout = 28800
wait_timeout = 28800
skip-ssl
-------------------------------------------------------------------------------------------------------------------

systemctl start mysqld
systemctl status mysqld
systemctl enable mysqld
grep 'temporary password' /var/log/mysqld.log
#查看MySQL安装时的默认随机密码
mysql -u root -p'随机密码'
ALTER USER 'root'@'localhost' IDENTIFIED BY 'OMxZrf4_k';
#修改root用户密码
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'OMxZrf4_k' WITH GRANT OPTION;
#设置允许root用户远程登录
FLUSH PRIVILEGES;
#刷新权限
SELECT User, Host FROM user;
#检查root用户登录权限是不是%
```

三台机器的角色分别是，master，备用master，slave。mha4mysql-node需要在三台机器分别安装，mha4mysql-manager为了能够在master出现故障时，切换到备用master，所以安装在slave。

| 角色 | master             | Slave1             | Slave2                |
| :--- | ------------------ | ------------------ | --------------------- |
|      | msyql5.7.40        | msyql5.7.40        | msyql5.7.40           |
|      | mha4mysql-node0.58 | mha4mysql-node0.58 | mha4mysql-node0.58    |
|      |                    |                    | mha4mysql-manager0.58 |
|      | 10.23.188.107      | 10.23.188.91       | 10.23.188.92          |

### 配置MySQL

安装MySQL插件

```bash
#进入数据库执行如下命名，查询MySQL插件地址
mysql> show variables like '%plugin_dir%';
+---------------+--------------------------+
| Variable_name | Value                    |
+---------------+--------------------------+
| plugin_dir    | /usr/lib64/mysql/plugin/ |
+---------------+--------------------------+
mysql> install plugin rpl_semi_sync_master SONAME 'semisync_master.so';
mysql> install plugin rpl_semi_sync_slave SONAME 'semisync_slave.so';
mysql> show plugins;
# 或者
mysql> select * from information_schema.plugins;
#检查最下方是否有刚才安装的两个插件
```

#### 查看半同步相关信息

```mysql
mysql> show variables like '%rpl_semi_sync%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | OFF        |
| rpl_semi_sync_master_timeout              | 10000      |
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
| rpl_semi_sync_slave_enabled               | OFF        |
| rpl_semi_sync_slave_trace_level           | 32         |
+-------------------------------------------+------------+
```

可以看到半同步插件还是未启用的off状态，所以需要修改my.cnf配置文件（加入以下内容），如下：
master和slave1加入如下

```bash
log-error=/usr/local/mysql/data/mysqld.err
log-bin=mysql-bin
binlog_format=mixed
rpl_semi_sync_master_enabled=1
rpl_semi_sync_master_timeout=1000
rpl_semi_sync_slave_enabled=1
relay_log_purge=0
relay-log = relay-bin
relay-log-index = slave-relay-bin.index
```

slave2加入如下

```bash
log-bin = mysql-bin
relay-log = relay-bin
relay-log-index = slave-relay-bin.index
read_only = 1
rpl_semi_sync_slave_enabled=1
#由于slave2只是用来做一个slave主机，所以无需开启master的半同步
```

#### 定时清理中继日志

在配置主从复制中，由于主和备主这两台主机上设置了参数relay_log_purge=0（表示不自动清除中继日志），所以slave节点需要定期删除中继日志，建议每个slave节点删除中继日志的时间错开。

```bash
crontab -e
0 5 * * * /usr/local/bin/purge_relay_logs - -user=root --password=密码 --port=端口 --disable_relay_log_purge >> /var/log/purge_relay.log 2>&1
```

更改配置文件后，需要执行以下命令进行重启MySQL，使配置文件生效。

```bash
systemctl restart mysqld
```

查看半同步状态，确认已开启：

```mysql
mysql> show variables like '%rpl_semi_sync%';       #查看半同步是否开启
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | ON         |    #这个值要为ON
| rpl_semi_sync_master_timeout              | 1000       |
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
| rpl_semi_sync_slave_enabled               | ON         |    #这个值也要为ON。
| rpl_semi_sync_slave_trace_level           | 32         |
+-------------------------------------------+------------+
```

> - rpl_semi_sync_master_status ：显示主服务是异步复制模式还是半同步复制模式，ON为半同步；
> - rpl_semi_sync_master_clients ：显示有多少个从服务器配置为半同步复制模式；
> - rpl_semi_sync_master_yes_tx ：显示从服务器确认成功提交的数量
> - rpl_semi_sync_master_no_tx ：显示从服务器确认不成功提交的数量
> - rpl_semi_sync_master_tx_avg_wait_time ：事务因开启 semi_sync ，平均需要额外等待的时间
> - rpl_semi_sync_master_net_avg_wait_time ：事务进入等待队列后，到网络平均等待时间

#### 创建相关用户

```mysql
master主机操作如下：
grant replication slave on *.* to mharep@'10.23.188.%' identified by 'vfZ5u7o_H';
# 创建用于同步的用户
grant all on *.* to manager@'10.23.188.%' identified by 'vfZ5u7o_H';
# 创建用户mha的manager监控的用户
# 查看master二进制相关的信息
mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 744
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)

slave1主机操作如下：
grant replication slave on *.* to mharep@'10.23.188.%' identified by 'vfZ5u7o_H';
grant all on *.* to manager@'10.23.188.%' identified by 'vfZ5u7o_H';

slave2主机操作如下
由于slave2无需做备主，所以不用创建用于同步数据的账户
grant all on *.* to manager@'10.23.188.%' identified by 'vfZ5u7o_H';
```

#### 配置主从复制

以下操作需要在slave1和slave2主机上分别执行一次，以便同步master主机的数据。

```mysql
#指定master主机的相关信息
mysql> change master to
    -> master_host='10.23.188.107',
    -> master_port=3307,
    -> master_user='mharep',
    -> master_password='vfZ5u7o_H',
    -> master_log_file = 'mysql-bin.000001',     #这是在master主机上查看到的二进制日志名
    -> master_log_pos=744;     #同上，这是查看到的二进制日志的position
Query OK, 0 rows affected, 2 warnings (0.01 sec)

mysql> start slave;      #启动slave
Query OK, 0 rows affected (0.00 sec)
```

最后查看slave主机的状态
在master主机上查看半同步相关信息，会发现同步的client已经变成了2

```bash
安装基础依赖包
在所有机器上执行:
yum install -y perl-ExtUtils-CBuilder perl-ExtUtils-MakeMaker perl-CPAN perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager perl-Time-HiRes
```

### 安装MHA-node

### **注：需要MHA-node需要在所有节点安装（包括manager主机节点）**

```bash
#下载包
wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58.tar.gz
#安装
tar zxf mha4mysql-node-0.58.tar.gz 
cd mha4mysql-node-0.58/
perl Makefile.PL 
make && make install
```

**注：接下来的所有操作，如果没有特别标注，则只需要在manager主机节点上执行即可。**

### 安装MHA-manager

```bash
#下载包
wget https://github.com/yoshinorim/mha4mysql-manager/releases/download/v0.58/mha4mysql-manager-0.58.tar.gz
#安装
tar zxf mha4mysql-manager-0.58.tar.gz 
cd mha4mysql-manager-0.58/
perl Makefile.PL 
make && make install
```

### 创建相应目录及复制所需文件

```bash
mkdir /etc/masterha
mkdir -p /masterha/app1
mkdir /scripts
cp samples/conf/* /etc/masterha/
cp samples/scripts/* /scripts/
```

### 修改mha-manager配置文件

注：manager共有两个主要的配置文件，一个是通用默认的，一个是单独的。需要将默认通用的配置文件的内容清空，如下：

```bash
#清空默认的配置文件
> /etc/masterha/masterha_default.cnf
```

```bash
cat> /etc/masterha/app1.cnf <<EPF
[server default]
manager_workdir=/masterha/app1   #指定工作目录
manager_log=/masterha/app1/manager.log    #指定日志文件
user=root    #指定manager管理数据库节点所使用的用户名
password=OMxZrf4_k    #对应的是上面用户的密码
ssh_user=root      #指定配置了ssh免密登录的系统用户
repl_user=mharep    #指定用于同步数据的用户名
repl_password=vfZ5u7o_H   #对应的是上面同步用户的 密码
ping_interval=1   #设置监控主库，发送ping包的时间间隔，默认是3秒，尝试三次没有回应时自动进行切换
master_ip_failover_script=/scripts/master_ip_failover
[server1]
hostname=10.23.188.91
port=3307
master_binlog_dir=/home/mysql_data/
candidate_master=1   #设置为候选master

[server2]
hostname=10.23.188.107
port=3307
master_binlog_dir=/home/mysql_data/
candidate_master=1   #设置为候选master，设置该参数后，发生主从切换以后将会将此库提升为主库

[server3]
hostname=10.23.188.92
port=3307
master_binlog_dir=/home/mysql_data/
no_master=1   #设置的不为备选主库
EPF
```

注意：MHA在读取配置文件时，不会忽略注释，所以，看完了看懂了把注释删除。
验证SSH有效性：

> masterha_check_ssh --global_conf=/etc/masterha/masterha_default.cnf --conf=/etc/masterha/app1.cnf

验证集群复制的有效性（MySQL必须都启动）：

> masterha_check_repl --global_conf=/etc/masterha/masterha_default.cnf --conf=/etc/masterha/app1.cnf

启动masterha_manager ：

> nohup masterha_manager --conf=/etc/masterha/app1.cnf --remove_dead_master_conf --ignore_last_failover &> /var/log/mha_manager.log &

> `--ignore_last_failover` 忽略上次切换。`MHA`每次故障切换后都会生成一个`app1.failover.complete`这样的文件，如果不加这个参数，需要删除这个文件才能再次启动

### 配置VIP

vip配置可以采用两种方式，一种通过keepalived的方式管理虚拟ip的浮动；另外一种通过脚本方式启动虚拟ip的方式（即不需要keepalived或者heartbeat类似的软件）。为了防止脑裂发生，推荐生产环境采用脚本的方式来管理虚拟ip，而不是使用keepalived来完成。

#### 创建一个VIP

ens192是机器的网卡名，根据具体情况写。

```bash
/sbin/ifconfig ens192:1 10.23.188.96/24 netmask 255.255.255.0 up
```

注意：vip需要设置一个，当前网络内没有其他人用的ip

#### 关闭一个vip

```bash
sudo /sbin/ifconfig eth0:1 down
```

#### 编写脚本实现虚拟漂移

```perl
#!/usr/bin/env perl
use strict;
use warnings FATAL => 'all';

use Getopt::Long;

my (
    $command,          $ssh_user,        $orig_master_host, $orig_master_ip,
    $orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
);

my $vip = '10.23.188.96/24';
my $key = '1';
my $ssh_start_vip = "/sbin/ifconfig ens192:$key $vip";
my $ssh_stop_vip = "/sbin/ifconfig ens192:$key down";

GetOptions(
    'command=s'          => \$command,
    'ssh_user=s'         => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s'   => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'new_master_host=s'  => \$new_master_host,
    'new_master_ip=s'    => \$new_master_ip,
    'new_master_port=i'  => \$new_master_port,
);

exit &main();

sub main {

    print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";

    if ( $command eq "stop" || $command eq "stopssh" ) {

        my $exit_code = 1;
        eval {
            print "Disabling the VIP on old master: $orig_master_host \n";
            &stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "start" ) {

        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n";
            &start_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "status" ) {
        print "Checking the Status of the script.. OK \n";
        exit 0;
    }
    else {
        &usage();
        exit 1;
    }
}

sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
sub stop_vip() {
     return 0  unless  ($ssh_user);
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}

sub usage {
    print
    "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}
```

这个脚本在软件包里自带，前面的步骤已经将example拷贝到指定目录，修改成对应的网卡和vip，记得加上执行权限。

### 测试

测试方法：停掉当前的master数据库，然后在slave数据库里执行show slave status\G，看当前的master节点和vip是否已经切换。

### 踩坑总结

> 当有slave节点宕机时，manager服务是无法启动的，建议在配置文件中暂时注释掉宕机节点的信息，待修复后再取消注释。

> 1. vip 自动从原来的 master 切换到新的 master,同时,manager 节点的监控进程自动退出。（正常退出，可以配置切换时发送邮件）
> 2. 在日志目录(/masterha/app1/)产生一个 app1.failover.complete 文件
> 3. /etc/masterha/app1.cnf 配置文件中原来老的 master 配置被删除。
> 4. 所以要将恢复后的主节点，配置文件重写。
> 5. 启动MySQL，然后再检查同步状态
> 6. 失败，报错，因为server_id得改，集群里，server_id必须唯一。

> 初次安装MySQL，通过rpm -hiv安装官网下载的rpm包，然后再检查同步状态时，报[error][/usr/share/perl5/vendor_perl/MHA/ServerManager.pm, ln301] install_driver(mysql) failed: Attempt to reload DBD/mysql.pm aborted.这是因为MHA是由perl语言开发的，perl操作数据库需要相应的驱动，按照步骤安装驱动。
>
> ```
> yum install -y cpan
> cpan -D DBI     
> [yes---sudo]
> cpan DBD::mysql
> ```

> 装完驱动发现又报cpan DBD::mysql Can't exec "mysql_config": 没有那个文件或目录
>这个文件MySQL安装好了应该自带的，通过rpm安装的5.7.36，居然没有，佛了，只能卸载掉，重新换一种姿势安装。
> 
>1. 首先通过rpm安装官方源，然后通过yum安装。（也就是本文记录的方式）
> 2. 然后发现根本没有驱动的问题了，也没有报找不到mysql_config。

> 问题又来了，安装的MySQL，yum安装完能正常启动，修改my.cnf之后启动就失败了(报错，data_dir非空，我把整个路径都删除了，启动还是继续报)。
>
> ```bash
> [client]
> port = 3307
> socket = /tmp/mysql.sock
> #character-set-server = utf8
> [mysqld]
> port = 3307
> socket = /var/run/mysqld/mysqld.sock
> pid-file = /var/run/mysqld/mysqld.pid
> user = mysql
> bind-address = 0.0.0.0
> server-id = 3
> skip-external-locking 
> explicit_defaults_for_timestamp
> datadir = /home/mysql_data
> init-connect = 'SET NAMES utf8'
> #character-set-server = utf8
> #skip-name-resolve
> back_log = 300
> max_connections = 1000
> max_connect_errors = 6000
> open_files_limit = 65535
> table_open_cache = 128
> max_allowed_packet = 4M
> binlog_cache_size = 1M
> max_heap_table_size = 8M
> tmp_table_size = 16M
> read_buffer_size = 2M
> read_rnd_buffer_size = 8M
> sort_buffer_size = 8M
> join_buffer_size = 8M
> #key_buffer_size = 4M
> thread_cache_size = 8
> query_cache_type = 1
> query_cache_size = 8M
> query_cache_limit = 2M
> ft_min_word_len = 4
> performance_schema = 0
> explicit_defaults_for_timestamp
> lower_case_table_names = 1
> skip-external-locking
> default_storage_engine = InnoDB
> innodb_file_per_table = 1
> innodb_open_files = 500
> innodb_buffer_pool_size = 128M
> innodb_write_io_threads = 4
> innodb_read_io_threads = 4
> innodb_thread_concurrency = 0
> innodb_purge_threads = 1
> innodb_flush_log_at_trx_commit = 2
> innodb_log_buffer_size = 2M
> innodb_log_file_size = 32M
> innodb_log_files_in_group = 3
> innodb_max_dirty_pages_pct = 90
> innodb_lock_wait_timeout = 120
> bulk_insert_buffer_size = 8M
> myisam_sort_buffer_size = 8M
> myisam_max_sort_file_size = 10G
> interactive_timeout = 28800
> wait_timeout = 28800
> skip-ssl
> log-error=/home/mysql_data/mysqld.err
> log-bin = mysql-bin
> relay-log = relay-bin
> relay-log-index = slave-relay-bin.index
> read_only = 1
> rpl_semi_sync_slave_enabled=1
> ```
>
> 最终修改my.cnf，将声明data_dir的一行，放到声明sock文件和pid文件路径之后MySQL终于启动正常了。(深坑)

### 参考资料

[MySQL高可用之MHA部署_Ray的技术博客_51CTO博客](https://blog.51cto.com/u_14154700/2472806)
[MHA 通过主从复制 实现高可用-XPBag](http://xpbag.com/97.html#一、mysql主从复制)
[MySQL 高可用架构 之 MHA (Centos 7.5 MySQL 5.7.18 MHA 0.58) - 司家勇 - 博客园](https://www.cnblogs.com/winstom/p/11022014.html#简介)
[mysql - MySQL集群搭建(5)-MHA高可用架构_个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000017486693#item-2-6)
[MySQL 高可用 MHA](https://gohalo.me/post/mysql-replication-mha.html)
[MySQL高可用架构之MHA - yayun - 博客园](https://www.cnblogs.com/gomysql/p/3675429.html)
[MHA搭建 - 简书](https://www.jianshu.com/p/7d84696e7d98)
