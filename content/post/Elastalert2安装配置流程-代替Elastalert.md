---
title: Elastalert2安装配置流程-代替Elastalert
slug: elastalert-2-installation-configuration-process-replaces-elastalert
tags: [ ELK, Alert ]
date: 2022-10-13T00:06:17+08:00
---

之前已经有elastalert的安装配置文档了，虽然经过了测试但是发现部分内置命令无法正常运行，并且不支持新版本kibana，代码仓库长时间没有更新，于是换用新版本的elastalert2.<!--more-->
[Requirements](https://elastalert2.readthedocs.io/en/latest/running_elastalert.html#requirements)

- Elasticsearch 7.x or 8.x, or OpenSearch 1.x or 2.x
- ISO8601 or Unix timestamped data
- Python 3.10. Require OpenSSL 1.1.1 or newer.
- pip

根据官方文档描述，需要升级Python环境到3.10，OPenSSL到1.1.1,事实上在Python3.7之后的版本，依赖的openssl，必须要是1.1或者1.0.2之后的版本。

### 开始安装Python3.10和OpenSSL

```bash
sudo yum -y groupinstall "Development tools"
sudo yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel libffi-devel gdbm-devel db4-devel libpcap-devel xz-devel make
sudo yum install zlib* -y
sudo yum install -y gcc gcc-c++ python-devel wget
sudo yum install -y zlib zlib-dev openssl-devel sqlite-devel bzip2-devel libffi libffi-devel gcc gcc-c++
wget https://www.openssl.org/source/openssl-1.1.1k.tar.gz
./config --prefix=/usr/local/openssl shared zlib 
sudo make && make install
mv /usr/bin/openssl /usr/bin/openssl.bak #备份当前openssl
mv /usr/include/openssl /usr/include/openssl.bak 
ln -s /usr/local/openssl/bin/openssl /usr/bin/openssl #配置使用新版本
ln -s /usr/local/openssl/include/openssl /usr/include/openssl
echo /usr/local/openssl/lib >> /etc/ld.so.conf   #更新动态链接库数据并重新加载
ldconfig -v
openssl version #查看是否升级成功
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

需要修改`vim /tmp/softwarebak/Python-3.10.4/Modules/Setup`在末尾加入一下内容

```
_socket socketmodule.c
# Socket module helper for SSL support; you must comment out the other
# socket line above, and possibly edit the SSL variable:
SSL=/usr/local/openssl
_ssl _ssl.c \
        -DUSE_SSL -I$(SSL)/include -I$(SSL)/include/openssl \
        -L$(SSL)/lib -lssl -lcrypto
```

### 重新编译安装Python

安装elastalert2具体流程，请看[官方文档](https://elastalert2.readthedocs.io/en/latest/running_elastalert.html#as-a-python-package)
