---
title: Nginx 配置 SSL 证书、黑白名单
slug: nginx-configuration-ssl-certificate,-black-and-white-list
tags: [Nginx, Linux]
date: 2022-07-12T22:57:28+08:00
---

HTTP 协议由于其简单快速、占用资源少，是一种用于分布式、协作式和超媒体信息系统的应用层协议，是互联网数据通信的基础，一直被用于网站服务器和浏览器之间进行数据传输。然而，HTTP 是明文协议，不会对数据进行任何方式的加密，在数据传输过程中存在明显安全隐患。

当黑客攻击并窃取了网站服务器和浏览器之间的传输报文时，可以直接读取传输的信息，造成网站或用户资料的泄露。因此，HTTP 不适用于敏感信息的传播，此时需要引入 HTTPS（超文本传输安全协议）。

### HTTPS

HTTPS（Hypertext Transfer Protocol Secure）是一种以计算机网络安全通信为目的的传输协议。它在 HTTP 基础上加入了 SSL/TLS 层，从而具备保护数据交换的隐私性、完整性和对网站服务器身份认证的功能。简单来说，HTTPS 就是安全版的 HTTP。

### 在 Nginx 服务器上安装证书

您可以将已签发的 SSL 证书安装到 Nginx 或 Tengine 服务器上。本文介绍如何下载 SSL 证书并在 Nginx 或 Tengine 服务器上完成配置。

编辑 Nginx 配置文件（`nginx.conf`），修改与证书相关的配置。

```bash
# 以下属性中，以 ssl 开头的属性表示与证书配置有关。
server {
    listen 443 ssl;
    # 配置 HTTPS 的默认访问端口为 443。
    # 如果未在此处配置 HTTPS 的默认访问端口，可能会导致 Nginx 无法启动。
    # 如果您使用 Nginx 1.15.0 及以上版本，请使用 listen 443 ssl 代替 listen 443 和 ssl on。
    # 网上有些文档仍在使用 ssl on，该写法已被官方弃用。
    server_name yourdomain;
    root html;
    index index.html index.htm;
    ssl_certificate cert/yourdomain.com.crt;     # (服务器证书)
    ssl_certificate_key cert/yourdomain.com.key; # (私钥文件)
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    # 表示使用的加密套件类型。
    ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3; # 表示支持的 TLS 协议版本，请根据实际安全策略评估是否启用 TLSv1.1。
    ssl_prefer_server_ciphers on;
    location / {
        root html;  # Web 网站程序存放目录。
        index index.html index.htm;
    }
}
```

### 设置 HTTP 请求自动跳转 HTTPS

```bash
server {
    listen 80;
    server_name yourdomain; # 需将 yourdomain 替换为证书绑定的域名。
    rewrite ^(.*)$ https://$host$1 permanent; # 将所有 HTTP 请求重定向到 HTTPS。
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
    return 301 https://$server_name$request_uri;
}
```

其他 Web 服务器或其他语言的实现方式此处不赘述。

### 记录一个 SSL 配置生成器

**SSL Configuration Generator** <https://ssl-config.mozilla.org/>

### Nginx 的黑白名单

Nginx 提供了基于 IP 的简单访问控制功能。例如，若要禁止 `1.2.3.4` 这个 IP 地址访问服务器，可在配置文件中添加如下指令：

```bash
deny 1.2.3.4;
```

常见配置语法如下：

```bash
# 屏蔽单个 IP 访问
deny IP;
# 允许单个 IP 访问
allow IP;
# 屏蔽所有 IP 访问
deny all;
# 允许所有 IP 访问
allow all;
# 屏蔽整个段：从 123.0.0.1 到 123.255.255.254
deny 123.0.0.0/8;
# 屏蔽 IP 段：从 123.45.0.1 到 123.45.255.254
deny 124.45.0.0/16;
# 屏蔽 IP 段：从 123.45.6.1 到 123.45.6.254
deny 123.45.6.0/24;
```

其他 Web 服务器或其他语言的实现方式此处不赘述。

### SSL 证书格式

- `.csr`  
  Certificate Signing Request，即证书签名请求文件。证书申请者在生成私钥的同时生成 CSR 文件。提交给证书颁发机构（CA）后，CA 使用其根证书私钥签名生成公钥证书文件（即颁发给用户的证书）。安装时可忽略该文件。

- `.key`  
  私钥文件，与证书一一对应。

- `.crt` / `.cert` / `.cer`  
  可为二进制格式（DER）或文本格式（PEM），仅包含证书内容，不保存私钥。一般 Linux 系统使用 `.crt` 后缀，`.cer` 多用于 Windows 系统。

  此外，可将多级证书（如中间证书、根证书）合并到同一文件中，形成完整的证书链。

- `.pem`  
  PEM（Privacy Enhanced Mail）格式，通常为 Base64 编码的文本文件，可包含证书、私钥或完整证书链。在 Nginx 中常见为证书文件。

> 配置 SSL 证书时，必需的是 `.crt` 和 `.key` 文件。