---
title: Nginx 配置 ssl 证书、黑白名单
slug: nginx-configuration-ssl-certificate,-black-and-white-list
tags: [ Nginx, Linux ]
date: 2022-07-12T22:57:28+08:00
---

HTTP 协议由于其简单快速、占用资源少，是一种用于分布式、协作式和超媒体信息系统的应用层协议,是互联网数据通信的基础。一直被用于网站服务器和浏览器之间进行数据传输。HTTP是互联网数据通信的基础,但是在数据传输的过程中也存在很明显的问题，由于 HTTP 是明文协议，不会对数据进行任何方式的加密。<!--more-->

当黑客攻击窃取了网站服务器和浏览器之间的传输报文的时，可以直接读取传输的信息，造成网站、用户资料的泄密。因此 HTTP 不适用于敏感信息的传播，这个时候需要引入 HTTPS（超文本传输安全协议）。

### HTTPS

HTTPS（Hypertext Transfer Protocol Secure ）是一种以计算机网络安全通信为目的的传输协议。在HTTP下加入了SSL层，从而具有了保护交换数据隐私和完整性和提供对网站服务器身份认证的功能，简单来说它就是安全版的 HTTP 。

### 在Nginx服务器上安装证书

您可以将已签发的SSL证书安装到Nginx或Tengine服务器上。本文介绍如何下载SSL证书并在Nginx或Tengine服务器上安装证书。

编辑Nginx配置文件（nginx.conf），修改与证书相关的配置。

```bash
#以下属性中，以ssl开头的属性表示与证书配置有关。
server {
    listen 443 ssl;
    #配置HTTPS的默认访问端口为443。
    #如果未在此处配置HTTPS的默认访问端口，可能会造成Nginx无法启动。
    #如果您使用Nginx 1.15.0及以上版本，请使用listen 443 ssl代替listen 443和ssl on。
    #网上有些文档还在写ssl on 这个写法已经被官方弃用了。
    server_name yourdomain;
    root html;
    index index.html index.htm;
    ssl_certificate cert/yourdomain.com.crt     ##(服务器证书)  
    ssl_certificate_key cert/yourdomain.com.key   ##(私钥文件）
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    #表示使用的加密套件的类型。
    ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3; #表示使用的TLS协议的类型，您需要自行评估是否配置TLSv1.1协议。
    ssl_prefer_server_ciphers on;
    location / {
        root html;  #Web网站程序存放目录。
        index index.html index.htm;
    }
}
```

### 设置HTTP请求自动跳转HTTPS

```bash
server {
    listen 80;
    server_name yourdomain; #需要将yourdomain替换成证书绑定的域名。
    rewrite ^(.*)$ https://$host$1; #将所有HTTP请求通过rewrite指令重定向到HTTPS。
    location / {
        index index.html index.htm;
    }
}
```

### 或者可以这样

```bash
server {
    listen 80;
    server_name dev.wangsl.com;
    return  301 https://$server_name$request_uri;
}
```

还有其他网页服务器，其他语言的写法，不赘述了。

### 记录一个ssl配置生成器

**SSL Configuration Generator**  https://ssl-config.mozilla.org/

### nginx的黑白名单

nginx提供了简单的基于IP的访问控制功能，比如我们要禁止`1.2.3.4`这个IP地址访问服务器，可以在配置文件中这样写：

```bash
deny 1.2.3.4;
```

```bash
# 屏蔽单个ip访问
deny IP;
# 允许单个ip访问
allow IP;
# 屏蔽所有ip访问
deny all;
# 允许所有ip访问
allow all;
#屏蔽整个段即从123.0.0.1到123.255.255.254访问的命令
deny 123.0.0.0/8
#屏蔽IP段即从123.45.0.1到123.45.255.254访问的命令
deny 124.45.0.0/16
#屏蔽IP段即从123.45.6.1到123.45.6.254访问的命令
deny 123.45.6.0/24
```

还有其他网页服务器，其他语言的写法，不赘述了。

### SSL 证书格式

.csr 
Certificate Signing Request，即证书签名请求文件。证书申请者在生成私钥的同时也生成证书请求文件。把CSR文件提交给证书颁发机构后，证书颁发机构使用其根证书私钥签名就生成了证书公钥文件，也就是颁发给用户的证书。安装时可忽略该文件

.key 
私钥，与证书一一配对

 .crt   .cert   .cer 
可以是二进制格式(der)，可以是文本格式(pem)。只包含证书，不保存私钥。一般Linux使用.crt后缀，.cer是windows后缀。

此外，可以将多级证书导入同一个证书文件中，形成一个包含完整证书链的证书

.pem

 证书文件（可忽略该文件）

> 配置ssl证书时，必要的是crt和key
