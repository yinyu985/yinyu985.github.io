---
title: OpenResty+Keepalived 组建高可用集群
slug: openresty+keepalived-to-form-a-highly-available-cluster
tags: [Nginx, Linux]
date: 2022-09-15T22:41:57+08:00
---

OpenResty 是一个基于 [Nginx](https://zh.wikipedia.org/wiki/Nginx) 的 Web 平台，可以使用其 LuaJIT 引擎执行 [Lua](https://zh.wikipedia.org/wiki/Lua) 指令码。由章亦春建立。2011 年之前，它最初由 [淘宝网](https://zh.wikipedia.org/wiki/淘宝网) 赞助，2012 年至 2016 年主要由 [Cloudflare](https://zh.wikipedia.org/wiki/Cloudflare) 支援。自 2017 年起，主要得到 OpenResty 软体基金会和 OpenResty 公司的支援。OpenResty 旨在构建可延伸的 Web 应用、Web 服务和动态 Web 闸道器。OpenResty 的架构是基于几个 nginx 模组，这些模组已经被扩充，以便将 nginx 扩充为一个 web 应用服务器，处理大量的请求。<!--more-->

### OpenResty 下载安装包

```bash
cd /home/resource && wget https://openresty.org/download/openresty-1.19.9.1.tar.gz  # 下载安装包到 /home/resource
```

### OpenResty 所需依赖的包安装

```bash
yum install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel -y
yum install -y openldap-devel
yum install -y git
```

### 下载第三方模块

```bash
mkdir /opt/extra_modules && cd /opt/extra_modules/  # 创建 extra_modules，用于存放第三方模块
cd /opt/extra_modules/ && git clone https://github.com/kvspb/nginx-auth-ldap.git   # 支持 ldap 网页认证
cd /opt/extra_modules/ && wget https://bitbucket.org/nginx-goodies/nginx-sticky-module-ng/get/master.tar.gz  # 支持 cookie
tar -xf master.tar.gz
mv nginx-goodies-nginx-sticky-module-ng-08a395c66e42/ nginx-sticky-module  # 把下载到的文件夹改成 nginx-sticky-module，方便后面编译，不然会报错：./configure: error: no /opt/extra_modules/nginx-sticky-module/config was found
cd /opt/extra_modules && git clone https://github.com/FRiCKLE/ngx_cache_purge.git  # 清除缓存
cd /opt/extra_modules && git clone https://github.com/yaoweibin/nginx_upstream_check_module.git   # 用于 upstream 健康检查
```

### 解压 OpenResty 安装包

```bash
cd /home/resource
tar -xf openresty-1.19.9.1.tar.gz
```

### 编译需要安装的模块

```bash
cd /home/resource/openresty-1.19.9.1
# 配置文件
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
--add-module=/opt/extra_modules/nginx_upstream_check_module \
-j8  # 用全部的两个核心去编译，我们的机器就俩核心
```

### 编译安装

```bash
gmake && gmake install
/home/openresty/nginx/sbin/nginx -t  # 查看新修改的 nginx 配置文件是否有语法错误/结构错误，如果能运行，证明编译安装成功
/home/openresty/nginx/sbin/nginx -V   # 显示所有的编译模块
```

### 创建可执行文件软链接

```bash
ln -s /home/openresty/nginx/sbin/nginx /usr/bin/
# ln 创建链接；-s 创建软链接
```

### 写入 Nginx 配置文件

```bash
vim /home/openresty/nginx/conf/nginx.conf
```

```nginx
# 定义作为 web 服务器/反向代理服务器时的 worker process 进程数
worker_processes    auto;
# 开启多核支持，且自动根据 CPU 个数均匀分配 worker process 进程数
worker_cpu_affinity auto;
# 指定一个 nginx 进程可以打开的最多文件描述符数目
worker_rlimit_nofile    65535;
# error_log 配置，等级类型：[ debug | info | notice | warn | error | crit ]
error_log  /var/log/nginx/error.log  debug;
# nginx 的进程 pid 位置
pid        /var/run/nginx.pid;
# 连接处理相关设置
events {
    # 使用 epoll 的 I/O 模型，必开项，极其有利于性能
    use            epoll;
    # 设置是否允许一个 worker 可以接受多个请求，默认是 off
    # 值为 OFF 时，一个 worker process 进程一次只接收一个请求，由 master 进程自动分配 worker（nginx 精于此道，故建议设置为 off）
    # 值为 ON 则一次可接收所有请求，可避免 master 进程额外调度，但是在高瞬时值的情况下可能导致 tcp flood
    multi_accept off;
    # 每个工作进程的并发连接数（默认为 1024）
    # 理论上 nginx 最大连接数 = worker_processes * worker_connections
    worker_connections 65535;
}

http {
    # mime.types 指定了 nginx 可以接受的 Content-Type，该文件默认位于 nginx.conf 的同级目录
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
    # 不记录 404 错误的日志
    log_not_found   off;
    # 隐藏 nginx 版本号
    server_tokens   off;
    # 开启 0 拷贝，提高文件传输效率
    sendfile    on;
    # 配合 sendfile 使用，启用后数据包会累计到一定大小之后才会发送，减小额外开销，提高网络效率
    tcp_nopush  on;
    # 启用后表示禁用 Nagle 算法，尽快发送数据
    # 与 tcp_nopush 结合使用的效果是：先填满包，再尽快发送
    # Nginx 只会针对处于 keep-alive 状态的 TCP 连接才会启用 tcp_nodelay
    tcp_nodelay on;
    # 指定客户端与服务端建立连接后发送 request body 的超时时间，超时 Nginx 将返回 http 408
    client_body_timeout 10;
    # 开启从 client 到 nginx 的连接长连接支持，指定每个 TCP 连接最多可以保持多长时间
    # keepalive_timeout 的值应该比 client_body_timeout 大
    keepalive_timeout   60;
    # keepalive_requests 指令用于设置一个 keep-alive 连接上可以服务的请求的最大数量，当最大请求数量达到时，连接将被关闭
    keepalive_requests  1000;
    # 客户端请求头部的缓冲区大小，设置等于系统分页大小即可，如果 header 过大可根据实际情况调整
    # 查看系统分页：getconf PAGESIZE
    client_header_buffer_size       32k;
    # 设置客户端请求的 Header 头缓冲区大小，如果客户端的 Cookie 信息较大，按需增加
    large_client_header_buffers     4 64k;
    # 优化读取 $request_body 变量时的 I/O 性能
    client_body_in_single_buffer    on;
    # 设定 request body 的缓冲大小，仅在 Nginx 被设置成使用内存缓冲时有效（使用文件缓冲时该参数无效）
    client_body_buffer_size     128k;
    # 开启 proxy 忽略客户端中断，避免 499 错误
    proxy_ignore_client_abort       on;
    # 默认的情况下 nginx 引用 header 变量时不能使用带下划线的变量，设置 underscores_in_headers 为 on 取消该限制
    underscores_in_headers      on;
    # 默认的情况下 nginx 会忽略带下划线的变量，设置 ignore_invalid_headers 为 off 取消该限制
    ignore_invalid_headers      off;
    # 设置客户端向服务端发送一个完整的 request header 的超时时间，优化弱网场景下 nginx 的性能
    client_header_timeout   10;
    # 设置向客户端传输数据的超时时间
    send_timeout        60;
    # 用于启用文件功能时用限制文件大小
    client_max_body_size    50m;
    # 文件压缩配置，对文本文件效果较好，对图像类应用效果一般反而徒增服务器资源消耗
    gzip        on;
    # 兼容 http 1.0
    gzip_http_version   1.0;
    # 压缩比，数值越大：压缩的程度越高、空间占用越低、压缩效率越低、资源消耗越大
    gzip_comp_level 6;
    # 设置压缩门限，小于该长度将不会进行压缩动作（数据过小的情况下，压缩效果不明显）
    gzip_min_length 1k;
    # 用于在 nginx 作为反向代理时，根据请求头中的“Via”字段决定是否启用压缩功能，默认值为 off，any 表示对所有请求启动压缩
    gzip_proxied    any;
    # 用于在启动 gzip 压缩功能时，在 http 响应中添加 Vary: Accept-Encoding 头字段告知接收方使用了 gzip 压缩
    gzip_vary       on;
    # 当 Agent 为 IE6 时禁用压缩：IE6 对 Gzip 不友好，所以不压缩
    gzip_disable    msie6;
    # 设置系统用于存储 gzip 的压缩结果数据流的缓存大小（4 4k 代表以 4k 为单位，按照原始数据大小以 4k 为单位的 4 倍申请内存）
    gzip_buffers    4 64k;
    # 指定需要压缩的文件 mime 类型
    gzip_types      text/xml text/plain text/css application/javascript application/x-javascript application/xml application/json application/rss+xml;
    # 作为反向代理服务器配置
    # 当请求未携带“Host”请求头时将 Host 设置为虚拟主机的主域名
    proxy_set_header        Host $host;
    # 设置真实客户端 IP
    proxy_set_header        X-Real-IP $remote_addr;
    # 简称 XFF 头，即 HTTP 的请求端真实的 IP，在有前置 CDN 或者负载均衡可能会被修改；如果要提取客户端真实 IP，需要根据实际情况调整，如若后端程序获得对 X-Forwarded-For 兼容性不好的话（没有考虑到 X-Forwarded-For 含有多个 IP 的情况），建议设置为：$http_x_forwarded_for
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    # 启用 nginx 和后端 server（upstream）之间长连接支持（必设项，否则很影响 nginx 性能），HTTP 协议中从 1.1 版本才支持长连接；启用时需要评估 upstream 的 keepalive 参数（默认是关闭的，比较懒的同学可以设置为 500）
    proxy_http_version 1.1;
    # 为了兼容老的协议以及防止 http 头中有 Connection close 导致的 keepalive 失效，需要及时清掉 HTTP 头部的 Connection
    # 该参数决定了访问完成后，后端 server 后如何处理本次连接，默认配置是主动 close（会给后端 server 带来大量的 TIME_WAIT 连接，降低后端 server 性能），设置为 "" 结合 proxy_http_version 设置连接保持（长连接）
    proxy_set_header Connection "";
    # 用于对发送给客户端的 URL 进行修改，使用不到的话可以关闭
    proxy_redirect          off;
    # 设置缓冲区的大小和数量，用于放置被代理的后端服务器取得的响应内容
    proxy_buffers           64 8k;
    # 设置和后端建立连接的超时时间，单位秒
    proxy_connect_timeout   60;
    # 设置 Nginx 向后端被代理服务器发送 read 请求后，等待响应的超时时间，默认 60 秒
    proxy_read_timeout 60;
    # 设置 Nginx 向后端被代理服务器发送 write 请求后，等待响应的超时时间，默认 60 秒
    proxy_send_timeout 60;
    # 用于配置存放 HTTP 报文头的哈希表容量，默认为 512 个字符。一般都设置为 1024，这个大小是哈希表的总大小
    # 设定了这个参数 Nginx 不是一次全部申请出来，需要用的时候才会申请
    # 但是当真正需要使用的时候也不是一次全部申请，而是会设置一个单次申请最大值（proxy_headers_hash_bucket_size）
    proxy_headers_hash_max_size 1024;
    # 用于设置 Nginx 服务器申请存放 HTTP 报文头的哈希表容量的单位大小，默认为 64 个字符。一般配置为 128
    # 这个大小是单次申请最多申请多大，也就是每次用需要申请，但是每次申请最大申请多少，整个哈希表大小不可超过上面设置的值
    proxy_headers_hash_bucket_size 128;
    # 预防 DDOS 攻击配置策略
    # limit_req_zone          $binary_remote_addr  zone=req:20m   rate=3r/s;
    # limit_req               zone=req  burst=60;
    # limit_zone              conn $binary_remote_addr  20m;
    # limit_conn              conn 5;
    # limit_rate              50k;
    # 设置 nginx 可以捕获的服务器名字（server_name）的最大数量
    server_names_hash_max_size    1024;
    # 设置 nginx 中 server_name 支持的最大长度
    server_names_hash_bucket_size 128;
    include conf.d/*.conf;
}
```

### 配置成服务，设置开机启动

```bash
cat > /etc/systemd/system/nginx.service << EOF
[Unit]
Description=Nginx(OpenResty) - high performance web server
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

### 重载 systemctl & 启用 Nginx 服务

```bash
systemctl daemon-reload && systemctl enable nginx
```

### 下载解压 keepalived、编译安装

```bash
cd /home/resource/
wget --no-check-certificate https://www.keepalived.org/software/keepalived-2.2.2.tar.gz
tar -xf keepalived-2.2.2.tar.gz
yum install -y gcc openssl-devel popt-devel  # 安装编译依赖
cd keepalived-2.2.2
./configure --prefix=/home/keepalived/
make && make install
mkdir /etc/keepalived
# 在安装好 keepalived 后有提供许多的配置文件模板（在 keepalived/etc 中）。启动 Keepalived 时默认会在 /etc/keepalived 目录中去找 keepalived.conf 文件，如果没有将配置文件放在该目录，启动 Keepalived 时需要使用 -f 选项来指定配置文件路径
cp /home/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
cp /home/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
ln -s /home/keepalived/sbin/keepalived /usr/bin/
```

### 修改 keepalived 配置文件

```bash
vim /etc/keepalived/keepalived.conf
```

### master

```bash
cat > /etc/keepalived/keepalived.conf << EOF
global_defs {
    router_id red_master137  # 一个没重复的名字即可，主从不可重复
}

vrrp_script nginx_check { 
    script "/etc/keepalived/nginx_check.sh"  # 检测 nginx 是否运行
    interval 1  # 脚本执行间隔，每 1s 检测一次
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    # nopreempt  # 设置 nopreempt 防止抢占资源，即使 master 恢复了，也不会去抢，只有 slave 挂了，才到 master
    interface ens192
    virtual_router_id 139  # 局域网中有相同的 virtual_router_id 会失败
    priority 100      # 权重，master 要大于 slave
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
cat > /etc/keepalived/keepalived.conf << EOF
global_defs {
    router_id red_slave138  # 一个没重复的名字即可，主从不可重复
}

vrrp_script nginx_check {
    script "/etc/keepalived/nginx_check.sh"  # 检测 nginx 是否运行
    interval 1  # 脚本执行间隔，每 1s 检测一次
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    # nopreempt  # 设置 nopreempt 防止抢占资源，即使 master 恢复了，也不会去抢，只有 slave 挂了，才到 master
    interface ens192
    virtual_router_id 139  # 局域网中有相同的 virtual_router_id 会失败
    priority 80      # 权重，master 要大于 slave
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

当本机 Nginx 停止时，停止本机 keepalived。

```bash
touch /etc/keepalived/nginx_check.sh
chmod +x /etc/keepalived/nginx_check.sh
```

```bash
#!/bin/bash
pidof nginx
if [ $? -ne 0 ]; then
    systemctl stop keepalived
fi
```

```bash
service keepalived start
# 配置开机自启动
systemctl enable keepalived
ps -aux | grep keepalived
```

### 测试

56、57 正常运行时，访问 VIP 时，显示 master。

> 图片没了自己脑补

停止 56，查看 keepalived 状态，也顺利的被脚本停掉了，再访问 VIP 时。

> 图片没了自己脑补

### 添加模块

测试成功。

假如用了一段时间，发现想要的功能忘了编译进去，可以重新编译。

```bash
[root@nginx1 /]# nginx -V  # 查看现有的 nginx 编译的参数，在此基础上添加
nginx version: openresty/1.19.9.1
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/home/openresty/nginx --with-cc-opt=-O2 --add-module=../ngx_devel_kit-0.3.1 --add-module=../iconv-nginx-module-0.14 --add-module=../echo-nginx-module-0.62 --add-module=../xss-nginx-module-0.06 --add-module=../ngx_coolkit-0.2 --add-module=../set-misc-nginx-module-0.32 --add-module=../form-input-nginx-module-0.12 --add-module=../encrypted-session-nginx-module-0.08 --add-module=../srcache-nginx-module-0.32 --add-module=../ngx_lua-0.10.20 --add-module=../ngx_lua_upstream-0.07 --add-module=../headers-more-nginx-module-0.33 --add-module=../array-var-nginx-module-0.05 --add-module=../memc-nginx-module-0.19 --add-module=../ngx_stream_lua-0.0.10 --with-ld-opt=-Wl,-rpath,/home/openresty/luajit/lib --pid-path=/var/run/nginx.pid --http-log-path=/var/log/nginx/access.log --error-log-path=/var/log/nginx/error.log --with-pcre --with-http_v2_module --with-http_ssl_module --with-pcre-jit --with-compat --with-threads --with-file-aio --with-http_gunzip_module --with-http_realip_module --with-http_gzip_static_module --with-http_degradation_module --with-http_auth_request_module --with-http_stub_status_module --without-http_memcached_module --without-mail_pop3_module --without-mail_imap_module --without-mail_smtp_module --add-module=/opt/extra_modules/nginx-auth-ldap --add-module=/opt/extra_modules/nginx-sticky-module --add-module=/opt/extra_modules/ngx_cache_purge --add-module=/opt/extra_modules/ngx-fancyindex --add-module=/opt/extra_modules/nginx_upstream_check_module --with-stream --with-stream_ssl_module --with-stream_ssl_preread_module
```

再编译除了需要加上这些和新的模块，还需要添加 `--with-luajit` 参数，由于再次编译时没有生成动态链接库，需要手动链接。不然编译完后是不能使用，提示 `libluajit-5.1.so.2` 找不到，我已经上过当了。

```bash
[root@nginx1 /]# nginx -V
nginx: error while loading shared libraries: libluajit-5.1.so.2: cannot open shared object file: No such file or directory
```

这就是重新编译时没有加 `--with-luajit` 参数的下场。

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
--with-luajit \  # 新加的
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
--add-module=/opt/extra_modules/ngx-fancyindex \  # 新加的
--add-module=/opt/extra_modules/nginx_upstream_check_module \
-j2  # 用全部的两个核心去编译，我们的机器就俩核心
```

然后 `make`，完成后 `/home/resource/openresty-1.19.9.1/build/nginx-1.19.9/objs` 这个路径下面会有一个 nginx 的可执行文件。备份好原来的，然后用这个替换。再用 `nginx -V` 检查新模块是不是已经加进来了。