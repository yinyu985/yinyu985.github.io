---
title: OpenResty+Keepalived 组建高可用集群
slug: openresty+keepalived-to-form-a-highly-available-cluster
tags: [ Nginx, Linux ]
date: 2022-09-15T22:41:57+08:00
---

OpenResty是一个基于[Nginx](https://zh.wikipedia.org/wiki/Nginx)的Web平台，可以使用其LuaJIT引擎执行[Lua](https://zh.wikipedia.org/wiki/Lua)指令码。由章亦春建立。2011年之前，它最初由[淘宝网](https://zh.wikipedia.org/wiki/淘宝网)赞助，2012年至2016年主要由[Cloudflare](https://zh.wikipedia.org/wiki/Cloudflare)支援。自2017年起，主要得到OpenResty软体基金会和OpenResty公司的支援。OpenResty旨在构建可延伸的Web应用、Web服务和动态Web闸道器。OpenResty的架构是基于几个nginx模组，这些模组已经被扩充，以便将nginx扩充为一个web应用服务器，处理大量的请求。<!--more-->

### OpenResty下载安装包

```bash
cd /home/resource && wget https://openresty.org/download/openresty-1.19.9.1.tar.gz   #下载安装包到/home/resource
```

### OpenResty所需依赖的包安装

```bash
yum install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel -y
yum install -y openldap-devel 
yum install -y git
```

### 下载第三方模块

```bash
mkdir /opt/extra_modules && cd /opt/extra_modules/  #创建extra_modules，用于存放第三方模块
cd /opt/extra_modules/ && git clone https://github.com/kvspb/nginx-auth-ldap.git   #支持ldap网页认证
cd /opt/extra_modules/ && wget https://bitbucket.org/nginx-goodies/nginx-sticky-module-ng/get/master.tar.gz #支持cookie
mv nginx-goodies-nginx-sticky-module-ng-08a395c66e42/ nginx-sticky-module  #把下载到的文件夹改成nginx-sticky-module，方便后面编译不然会报错./configure: error: no /opt/extra_modules/nginx-sticky-module/config was found
tar  -xf master.tar.gz 
cd /opt/extra_modules && git clone https://github.com/FRiCKLE/ngx_cache_purge.git  #清除缓存
cd /opt/extra_modules && git clone https://github.com/yaoweibin/nginx_upstream_check_module.git   #用于ustream健康检查
```

### 解压OpenResty安装包

```bash
cd /home/resource
tar -xf openresty-1.19.9.1.tar.gz
```

### 编译需要安装的模块

```bash
cd /home/resource/openresty-1.19.9.1
#配置文件
./configure --prefix=/home/openresty \
--pid-path=/var/run/nginx.pid \
--http-log-path=/var/log/nginx/access.log \
--error-log-path=/var/log/nginx/error.log \
--with-luajit \
--with-pcre \
--with-http_v2_module \
--with-http_ssl_module \
--with-pcre-jit \
--with-compat \
--with-threads \
--with-file-aio \
--with-http_gunzip_module \
--with-http_iconv_module \
--with-http_realip_module \
--with-http_gzip_static_module \
--with-http_degradation_module \
--with-http_auth_request_module \
--with-http_stub_status_module \
--without-lua_resty_memcached \
--without-http_memcached_module \
--without-lua_resty_mysql \
--without-lua_redis_parser \
--without-lua_resty_redis \
--without-http_redis_module \
--without-http_redis2_module \
--without-lua_rds_parser \
--without-http_rds_csv_module \
--without-http_rds_json_module \
--without-mail_pop3_module \
--without-mail_imap_module \
--without-mail_smtp_module \
--add-module=/opt/extra_modules/nginx-auth-ldap \
--add-module=/opt/extra_modules/nginx-sticky-module \
--add-module=/opt/extra_modules/ngx_cache_purge \
--add-module=/opt/extra_modules/nginx_upstream_check_module  \
-j8#用全部的两个核心去编译#我们的机器就俩核心
```

### 编译安装

```bash
gmake && gmake install
/home/openresty/nginx/sbin/nginx -t  #查看新修改的nginx配置文件是否有语法错误/结构错误，如果能运行，证明编译安装成功
/home/openresty/nginx/sbin/nginx -V    #显示所有的编译模块
```

### 创建可执行文件软链接

```bash
ln -s /home/openresty/nginx/sbin/nginx /usr/bin/
#ln创建链接；-s创建软链接
```

### 写入Nginx配置文件

```bash
vim /home/openresty/nginx/conf/nginx.conf  
```

```bash
# 定义作为web服务器/反向代理服务器时的 worder process 进程数
worker_processes    auto;
# 开启多核支持，且自动根据CPU个数均匀分配 worder process 进程数
worker_cpu_affinity auto;
# 指定一个nginx进程可以打开的最多文件描述符数目
worker_rlimit_nofile    65535;
# error_log配置，等级类型：[ debug | info | notice | warn | error | crit ]
error_log  /var/log/nginx/error.log  debug;
# nginx的进程pid位置；
pid        /var/run/nginx.pid;
# 连接处理相关设置
events{
    # 使用epoll的 I/O 模型，必开项，极其有利于性能
    use            epoll;
    # 设置是否允许一个worker可以接受多个请求，默认是off；
    # 值为OFF时，一个worker process进程一次只接收一个请求，由master进程自动分配worker（nginx精于此道，故建议设置为off）；
    # 值为ON则一次可接收所有请求，可避免master进程额外调度，但是在高瞬时值的情况下可能导致tcp flood；
    multi_accept off;
    # 每个工作进程的并发连接数（默认为1024）
    # 理论上nginx最大连接数 = worker_processes * worker_connections
    worker_connections 65535;
}
http {
    # mime.types 指定了nginx可以接受的 Content-Type，该文件默认位于nginx.conf的同级目录
    include       mime.types;
    # 设置默认文件类型，application/octet-stream 表示未知的应用程序文件，浏览器一般不会自动执行或询问执行
    default_type  application/octet-stream;
    # 设置日志的记录格式
    log_format main escape=json '{ "time": "$time_iso8601", '
        '"remote_addr": "$remote_addr", '
        '"status": "$status", '
        '"bytes_sent": "$bytes_sent", '
        '"host": "$host", '
        '"request_method": "$request_method", '
        '"request_uri": "$request_uri", '
        '"request_time": "$request_time", '
        '"response_time": "$upstream_response_time",'
        '"http_referer": "$http_referer", '
        '"body_bytes_sent": "$body_bytes_sent", '
        '"http_user_agent": "$http_user_agent", '
        '"http_x_forwarded_for": "$http_x_forwarded_for", '
        '"cookie": "$http_cookie" '
        '}';
    # 用来指定日志文件的存放路径及内容格式
    access_log  /var/log/nginx/access.log  main;
    # 不记录404错误的日志
    log_not_found   off;
    # 隐藏nginx版本号
    server_tokens   off;
    # 开启0拷贝，提高文件传输效率
    sendfile    on;
    # 配合 sendfile 使用，启用后数据包会累计到一定大小之后才会发送，减小额外开销，提高网络效率；
    tcp_nopush  on;
    # 启用后表示禁用 Nagle 算法，尽快发送数据
    # 与 tcp_nopush 结合使用的效果是：先填满包，再尽快发送
    # Nginx 只会针对处于 keep-alive 状态的 TCP 连接才会启用 tcp_nodelay
    tcp_nodelay on;
    # 指定客户端与服务端建立连接后发送 request body 的超时时间，超时Nginx将返���http 408
    client_body_timeout 10;
    # 开启从client到nginx的连接长连接支持，指定每个 TCP 连接最多可以保持多长时间
    # keepalive_timeout的值应该比 client_body_timeout 大
    keepalive_timeout   60;
    # keepalive_requests指令用于设置一个keep-alive连接上可以服务的请求的最大数量，当最大请求数量达到时，连接将被关闭
    keepalive_requests  1000;
    # 客户端请求头部的缓冲区大小，设置等于系统分页大小即可，如果header过大可根据实际情况调整；
    # 查看系统分页：getconf PAGESIZE
    client_header_buffer_size       32k;
    # 设置客户端请求的Header头缓冲区大小，如果客户端的Cookie信息较大，按需增加
    large_client_header_buffers     4 64k;
    # 优化读取$request_body变量时的I/O性能
    client_body_in_single_buffer    on;
    # 设定request body的缓冲大小，仅在 Nginx被设置成使用内存缓冲时有效（使用文件缓冲���该参数无效）
    client_body_buffer_size     128k;
    # 开启proxy忽略客户端中断，避免499错误
    proxy_ignore_client_abort       on;
    # 默认的情况下nginx引用header变量时不能使用带下划线的变量，设置underscores_in_headers为 on取消该限制
    underscores_in_headers      on;
    # 默认的情况下nginx会忽略带下划线的变量，设置ignore_invalid_headers为off取消该限制
    ignore_invalid_headers      off;
    # 设置客户端向服务端发送一个完整的 request header 的超时时间，优化弱网场景下nginx的性能
    client_header_timeout   10;
    # 设置向客户端传输数据的超时时间
    send_timeout        60;
    # 用于启用文件功能时用限制文件大小；
    client_max_body_size    50m;
    # 文件压缩配置，对文本文件效果较好，对图像类应用效果一般反而徒增服务器资源消耗
    gzip        on;
    # 兼容http 1.0
    gzip_http_version   1.0;
    # 压缩比，数值越大：压缩的程度越高、空间占用越低、压缩效率越低、资源消耗越大
    gzip_comp_level 6;
    # 设置压缩门限，小于该长度将不会进行压缩动作（数据过小的情况下，压缩效果不明显）
    gzip_min_length 1k;
    # 用于在nginx作为反向代理时，根据请求头中的“Via”字段决定是否启用压缩功能，默认值为off，any表示对所有请求启动压缩；
    gzip_proxied    any;
    # 用于在启动gzip压缩功能时，在http响应中添加Vary: Accept-Encoding头字段告知接收方使用了gzip压缩；
    gzip_vary       on;
    # 当Agent为IE6时禁用压缩：IE6对Gzip不友好，所以不压缩
    gzip_disable    msie6;
    # 设置系统用于存储gzip的压缩结果数据流的缓存大小（4 4k 代表以4k为单位，按照原始数据大小以4k为单位的4倍申请内存）
    gzip_buffers    4 64k;
    # 指定需要压缩的文件mime类型
    gzip_types      text/xml text/plain text/css application/javascript application/x-javascript application/xml application/json application/rss+xml;
    # 作为反向代理服务器配置
    # 当请求未携带“Host”请求头时将Host设置为虚拟主机的主域名
    proxy_set_header        Host $host;
    # 设置真实客户端IP
    proxy_set_header        X-Real-IP $remote_addr;
    # 简称XFF头，即HTTP的请求端真实的IP，在有前置cdn或者负载均衡可能会被修改；如果要提取客户端真实IP，需要根据实际情况调整，如若后端程序获得对X-Forwarded-For兼容性不好的话（没有考虑到X-Forwarded-For含有多个IP的情况），建议设置为：$http_x_forwarded_for
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    # 启用nginx和后端server（upstream）之间长连接支持（必设项，否则很影响nginx性能），HTTP协议中从1.1版本才支持长连接；启用时需要评估upstream的keepalive参数（默认是关闭的，比较懒的���学可以设置为500）
    proxy_http_version 1.1;
    # 为了兼容老的协议以及防止http头中有Connection close导致的keepalive失效，需要及时清掉HTTP头部的Connection；
    # 该参数决定了访问完成后，后端server后如何处理本次连接，默认配置是主动close（会给后端server带来大量的TIME_WAIT连接，降低后端server性能），设置为""结合proxy_http_version设置连接保持（长连接）；
    proxy_set_header Connection "";
    # 用于对发送给客户端的URL进行修改，使用不到的话可以关闭
    proxy_redirect          off;
    # 设置缓冲区的大小和数量，用于放置被代理的后端服务器取得的响应内容
    proxy_buffers           64 8k;
    # 设置和后端建立连接的超时时间，单位秒
    proxy_connect_timeout   60;
    # 设置Nginx向后端被代理服务器发送read请求后，等待响应的超时时间，默认60秒
    proxy_read_timeout 60;
    # 设置Nginx向后端���代理服务器发送write请求后，等待响应的超时时间，默认60秒
    proxy_send_timeout 60;
    # 用于配置存放HTTP报文头的哈希表容量，默认为512个字符。一般都设置为1024，这个大小是哈希表的总大小，
    #设定了这个参数Nginx不是一次全部申请出来，需要用的时候才会申请；
    #但是当真正需要使用的时候也不是一次全部申请，而是会设置一个单次申请最大值（proxy_headers_hash_bucket_size）
    proxy_headers_hash_max_size 1024;
    # 用于设置Nginx服务器申请存放HTTP报文头的哈希表容量的单位大小，默认为64个字符。一般配置为128。
    #这个大小是单次申请最多申请多大，也就是每次用需要申请，但是每次申请最大申请多少，整个哈希表大小不可超过上面设置的值。
    proxy_headers_hash_bucket_size 128;
    # 预防 DDOS 攻击配置策略
    #limit_req_zone          $binary_remote_addr  zone=req:20m   rate=3r/s;
    #limit_req               zone=req  burst=60;
    #limit_zone              conn $binary_remote_addr  20m;
    #limit_conn              conn 5;
    #limit_rate              50k;
    # 设置nginx可以捕获的服务器名字（server_name）的最大数量
    server_names_hash_max_size    1024;
    # 设置nginx中server_name支持的最大长度
    server_names_hash_bucket_size 128;
    include conf.d/*.conf;
}
```

### 配置成服务，设置开机启动

```bash
cat > /etc/systemd/system/nginx.service << EOF
[Unit]
Description=Nginx(OpenResty ) - high performance web server
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
User=root
Group=root
Type=forking
PIDFile=/var/run/nginx.pid
ExecStartPre=/home/openresty/nginx/sbin/nginx -t -c /home/openresty/nginx/conf/nginx.conf
ExecStart=/home/openresty/nginx/sbin/nginx -c /home/openresty/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

### 重载 systemctl&启用Nginx服务

```bash
systemctl daemon-reload && systemctl enable nginx
```

### 下载解压keepalived、编译安装

```bash
cd /home/resource/
wget --no-check-certificate https://www.keepalived.org/software/keepalived-2.2.2.tar.gz
tar -xf keepalived-2.2.2.tar.gz
yum install -y gcc openssl-devel popt-devel#安装编译依赖
cd keepalived-2.2.2
./configure --prefix=/home/keepalived/
make && make install
mkdir /etc/keepalived
#，在安装好keepalived后有提供许多的配置文件模板（在keepalived/etc中）。启动Keepalived时默认会在/etc/keepalived目录中去找keepalived.conf文件，如果没有将配置文件放在该目录，启动Keepalived时需要使用-f选项来指定配置文件路径：
cp /home/keepalived/etc/keepalived/keepalived.conf  /etc/keepalived/
cp /home/keepalived/etc/sysconfig/keepalived  /etc/sysconfig/
ln -s /home/keepalived/sbin/keepalived /usr/bin/
```

### 修改keepalived配置文件

```bash
vim /etc/keepalived/keepalived.conf
```

### master

```bash
cat >/etc/keepalived/keepalived.conf<<EOF
global_defs {
    router_id red_master137 #一个没重复的名字即可,主从不可重复
}

vrrp_script nginx_check { 
    script "/etc/keepalived/nginx_check.sh"  # 检测nginx是否运行
    interval 1 #脚本执行间隔，每1s检测一次
    weight 2
}
vrrp_instance VI_1 {
    state BACKUP
    #nopreempt  # 设置nopreempt防止抢占资源,即使master恢复了，也不会去抢，只有slave挂了，才到master
    interface ens192
    virtual_router_id 139 #局域网中有相同的virtual_router_id会失败
    priority 100      # 权重，master要大于slave
    advert_int 1      # 主备通讯时间间隔
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.23.188.139
    }
    track_script {
        nginx_check
    }
}
EOF
```

### slave

```bash
cat >/etc/keepalived/keepalived.conf<<EOF
global_defs {
    router_id red_slave138 #一个没重复的名字即可,主从不可重复
}

vrrp_script nginx_check {
    script "/etc/keepalived/nginx_check.sh" # 检测nginx是否运行
    interval 1 #脚本执行间隔，每1s检测一次
    weight 2
}
vrrp_instance VI_1 {
    state BACKUP
    #nopreempt  # 设置nopreempt防止抢占资源,即使master恢复了，也不会去抢，只有slave挂了，才到master
    interface ens192
    virtual_router_id 139 #局域网中有相同的virtual_router_id会失败
    priority 80      # 权重，master要大于slave
    advert_int 1       # 主备通讯时间间隔
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.23.188.139
    }
    track_script {
        nginx_check
    }
}
EOF
```

### 编写脚本停止脚本

当本机Nginx停止时，停止本机keepalived

```bash
touch /etc/keepalived/nginx_check.sh
chmod +x /etc/keepalived/nginx_check.sh

#!/bin/bash
pidof nginx
if [ $? -ne 0 ];then
systemctl stop keepalived
fi

##############################
service keepalived start
# 配置开机自启动
systemctl enable keepalived
ps -aux |grep keepalived
```

### 测试

56、57 正常运行时，访问VIP时，显示master

> 图片没了自己脑补

停止56，查看keepalived状态，也顺利的被脚本停掉了，再访问VIP时

> 图片没了自己脑补

### 添加模块

测试成功

假如用了一段时间，发现想要的功能忘了编译进去，可以重新编译。

```bash
[root@nginx1 /]# nginx -V  #查看现有的nginx编译的参数，在此基础上添加
nginx version: openresty/1.19.9.1
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/home/openresty/nginx --with-cc-opt=-O2 --add-module=../ngx_devel_kit-0.3.1 --add-module=../iconv-nginx-module-0.14 --add-module=../echo-nginx-module-0.62 --add-module=../xss-nginx-module-0.06 --add-module=../ngx_coolkit-0.2 --add-module=../set-misc-nginx-module-0.32 --add-module=../form-input-nginx-module-0.12 --add-module=../encrypted-session-nginx-module-0.08 --add-module=../srcache-nginx-module-0.32 --add-module=../ngx_lua-0.10.20 --add-module=../ngx_lua_upstream-0.07 --add-module=../headers-more-nginx-module-0.33 --add-module=../array-var-nginx-module-0.05 --add-module=../memc-nginx-module-0.19 --add-module=../ngx_stream_lua-0.0.10 --with-ld-opt=-Wl,-rpath,/home/openresty/luajit/lib --pid-path=/var/run/nginx.pid --http-log-path=/var/log/nginx/access.log --error-log-path=/var/log/nginx/error.log --with-pcre --with-http_v2_module --with-http_ssl_module --with-pcre-jit --with-compat --with-threads --with-file-aio --with-http_gunzip_module --with-http_realip_module --with-http_gzip_static_module --with-http_degradation_module --with-http_auth_request_module --with-http_stub_status_module --without-http_memcached_module --without-mail_pop3_module --without-mail_imap_module --without-mail_smtp_module --add-module=/opt/extra_modules/nginx-auth-ldap --add-module=/opt/extra_modules/nginx-sticky-module --add-module=/opt/extra_modules/ngx_cache_purge --add-module=/opt/extra_modules/ngx-fancyindex --add-module=/opt/extra_modules/nginx_upstream_check_module --with-stream --with-stream_ssl_module --with-stream_ssl_preread_module
```

再编译除了需要加上这些和新的模块，还需要添加--with-luajit参数，由于再次编译时没有生成动态链接库，需要手动链接。不然编译完后是不能使用,提示libluajit-5.1.so.2找不到，我已经上过当了。

```bash
[root@nginx1 /]# nginx -V
nginx: error while loading shared libraries: libluajit-5.1.so.2: cannot open shared object file: No such file or directory
```

这就是重新编译时没有加--with-luajit参数 的下场

```bash
./configure --prefix=/home/openresty \
--pid-path=/var/run/nginx.pid \
--http-log-path=/var/log/nginx/access.log \
--error-log-path=/var/log/nginx/error.log \
--with-luajit \
--with-pcre \
--with-http_v2_module \
--with-http_ssl_module \
--with-pcre-jit \
--with-compat \
--with-threads \
--with-file-aio \
--with-luajit \  #新加的
--with-http_gunzip_module \
--with-http_iconv_module \
--with-http_realip_module \
--with-http_gzip_static_module \
--with-http_degradation_module \
--with-http_auth_request_module \
--with-http_stub_status_module \
--without-lua_resty_memcached \
--without-http_memcached_module \
--without-lua_resty_mysql \
--without-lua_redis_parser \
--without-lua_resty_redis \
--without-http_redis_module \
--without-http_redis2_module \
--without-lua_rds_parser \
--without-http_rds_csv_module \
--without-http_rds_json_module \
--without-mail_pop3_module \
--without-mail_imap_module \
--without-mail_smtp_module \
--add-module=/opt/extra_modules/nginx-auth-ldap \
--add-module=/opt/extra_modules/nginx-sticky-module \
--add-module=/opt/extra_modules/ngx_cache_purge \
--add-module=/opt/extra_modules/ngx-fancyindex \ #新加的
--add-module=/opt/extra_modules/nginx_upstream_check_module \
-j2#用全部的两个核心去编译#我们的机器就俩核心
```

然后make，完成后/home/resource/openresty-1.19.9.1/build/nginx-1.19.9/objs这个路径下面会有一个nginx的可执行文件
备份好原来的，然后用这个替换。再用nginx -V检查新模块是不是已经加进来了。
