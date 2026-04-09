---
title: Virtualenv 和 Virtualenvwrapper 使用指南
slug: virtualenv-and-virtualenvwrapper-user's-guide
tags: [Python]
date: 2021-06-10T19:55:08+08:00
---

Python 的虚拟环境可以为一个 Python 项目提供独立的解释环境、依赖包等资源，既能够很好地隔离不同项目使用不同 Python 版本带来的冲突，而且还能方便项目的发布。<!--more-->

## virtualenv

[virtualenv](http://pypi.python.org/pypi/virtualenv) 可用于创建独立的 Python 环境，它会创建一个包含项目所必须要的执行文件。

### 安装 virtualenv

```bash
pip install virtualenv
```

### 配置 pip 安装第三方库的镜像源地址

我们都知道，国内连接国外的服务器都会比较慢，有时候下载经常出现超时的情况。这时可以尝试使用国内优秀的 [豆瓣源](https://pypi.douban.com/simple) 镜像来安装。

使用豆瓣源安装 virtualenv：

```bash
pip install -i https://pypi.douban.com/simple virtualenv
```

### virtualenv 使用方法

如下命令表示在当前目录下创建一个名叫 `env` 的目录（虚拟环境），该目录下包含了独立的 Python 运行程序，以及 pip 副本用于安装其他的 package：

```bash
virtualenv env
```

当然在创建 `env` 的时候可以选择 Python 解释器，例如：

```bash
virtualenv -p /usr/local/bin/python3 venv
```

默认情况下，虚拟环境会依赖系统环境中的 site packages，就是说系统中已经安装好的第三方 package 也会安装在虚拟环境中。如果不想依赖这些 package，那么可以加上参数 `--no-site-packages` 建立虚拟环境：

```bash
virtualenv --no-site-packages [虚拟环境名称]
```

### 启动虚拟环境

```bash
cd env
source ./bin/activate
```

注意此时命令行会多一个 `(env)`，`env` 为虚拟环境名称，接下来所有模块都只会安装到这个虚拟的环境中去。

### 退出虚拟环境

```bash
deactivate
```

如果想删除虚拟环境，那么直接运行 `rm -rf venv/` 命令即可。

### 在虚拟环境安装 Python packages

Virtualenv 附带有 pip 安装工具，因此需要安装的 packages 可以直接运行：

```bash
pip install [套件名称]
```

## Virtualenvwrapper

Virtualenvwrapper 是一个虚拟环境管理工具，它能够管理创建的虚拟环境的位置，并能够方便地查看虚拟环境的名称以及切换到指定的虚拟环境。

### 安装（确保 virtualenv 已安装）

```bash
pip install virtualenvwrapper
```

或者使用豆瓣源（Windows 用户可使用 `virtualenvwrapper-win`）：

```bash
pip install -i https://pypi.douban.com/simple virtualenvwrapper-win
```

> 注：安装需要在非虚拟环境下进行。

### 创建虚拟环境

```bash
mkvirtualenv env
```

创建虚拟环境完成后，会自动切换到创建的虚拟环境中。

当然也可以指定虚拟机的 Python 版本：

```bash
mkvirtualenv env -p C:\python27\python.exe
```

### 列出虚拟环境列表

```bash
workon
```

或

```bash
lsvirtualenv
```

### 启动/切换虚拟环境

使用 `workon [virtual-name]` 即可切换到对应的虚拟环境：

```bash
workon [虚拟环境名称]
```

### 删除虚拟环境

```bash
rmvirtualenv [虚拟环境名称]
```

### 离开虚拟环境

离开虚拟环境，和 virtualenv 一样的命令：

```bash
deactivate
```