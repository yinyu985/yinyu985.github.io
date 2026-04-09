---
title: Elastalert2 安装配置流程 - 代替 Elastalert
slug: elastalert-2-installation-configuration-process-replaces-elastalert
tags: [ELK]
date: 2022-10-13T00:06:17+08:00
---

之前已经有 elastalert 的安装配置文档了，虽然经过了测试但是发现部分内置命令无法正常运行，并且不支持新版本 Kibana，代码仓库长时间没有更新，于是换用新版本的 elastalert2。  
[Requirements](https://elastalert2.readthedocs.io/en/latest/running_elastalert.html#requirements)

- Elasticsearch 7.x 或 8.x，或 OpenSearch 1.x 或 2.x
- ISO8601 或 Unix timestamped 数据
- Python 3.10，Require OpenSSL 1.1.1 或更新版本
- pip

根据官方文档描述，需要升级 Python 环境到 3.10，OpenSSL 到 1.1.1。事实上在 Python 3.7 之后的版本，依赖的 OpenSSL 必须要是 1.1 或 1.0.2 之后的版本。

### 开始安装 Python 3.10 和 OpenSSL

```bash
sudo yum -y groupinstall "Development tools"
sudo yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel libffi-devel gdbm-devel db4-devel libpcap-devel xz-devel make
sudo yum install zlib* -y
sudo yum install -y gcc gcc-c++ python-devel wget
sudo yum install -y zlib zlib-dev openssl-devel sqlite-devel bzip2-devel libffi libffi-devel gcc gcc-c++
wget https://www.openssl.org/source/openssl-1.1.1k.tar.gz
./config --prefix=/usr/local/openssl shared zlib 
sudo make && make install
mv /usr/bin/openssl /usr/bin/openssl.bak
mv /usr/include/openssl /usr/include/openssl.bak
ln -s /usr/local/openssl/bin/openssl /usr/bin/openssl
ln -s /usr/local/openssl/include/openssl /usr/include/openssl
echo /usr/local/openssl/lib >> /etc/ld.so.conf
ldconfig -v
openssl version
wget https://www.python.org/ftp/python/3.10.4/Python-3.10.4.tgz
sudo ./configure --prefix=/usr/local/python3 --with-ssl=/usr/local/openssl
sudo make && sudo make install && sudo make clean
```

### 安装完成可能会出现

```bash
WARNING: pip is configured with locations that require TLS/SSL, however the ssl module in Python is not available.
Could not fetch URL https://pypi.org/simple/pip/: There was a problem confirming the ssl certificate: HTTPSConnectionPool(host='pypi.org', port=443): Max retries exceeded with url: /simple/pip/ (Caused by SSLError("Can't connect to HTTPS URL because the SSL module is not available.")) - skipping
```

### 修改完善

需要修改 `vim /tmp/softwarebak/Python-3.10.4/Modules/Setup`，在末尾加入以下内容：

```
_socket socketmodule.c
# Socket module helper for SSL support; you must comment out the other
# socket line above, and possibly edit the SSL variable:
SSL=/usr/local/openssl
_ssl _ssl.c \
        -DUSE_SSL -I$(SSL)/include -I$(SSL)/include/openssl \
        -L$(SSL)/lib -lssl -lcrypto
```

### 重新编译安装 Python

安装 elastalert2 具体流程，请看[官方文档](https://elastalert2.readthedocs.io/en/latest/running_elastalert.html#as-a-python-package)