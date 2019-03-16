---
title: Linux下安装python3环境
date: 2017-10-28 21:59:58
tags:
- python
categories: 
- linux
---

# 前言

最近听到ubuntu 17版本自带python3环境的消息，很是高兴，因为linux的本地支持python3有助于新版本python的推广，以前的版本都是不带python3环境的，许多人使用服务器还需要安装python3虚拟环境，这个是需要一些时间的，有时候因为某个依赖库没有安装上而抛出各种异常让人烦恼不已，下面介绍一下以前我安装python3开发环境的方法。

# 下载

首先下载python3的安装包

```
wget https://www.python.org/ftp/python/3.5.2/Python-3.5.2.tar.xz
```

# 解压安装

```
tar xJf Python-3.5.2.tar.xz
cd Python-3.5.2
./configure --prefix=/opt/python3
```

在解压安装中遇到异常，可尝试安装相关依赖环境

```
Ubuntu:apt-get install make build-essential libssl-dev zlib1g-dev libbz2-dev libsqlite3-dev
CentOS:yum install zlib-devel bzip2-devel sqlite sqlite-devel openssl-devel
```

# 链接python3到新的python3解释器

``` 
ln -s /opt/python3/bin/python3.5 /usr/local/bin/python3
```

# 安装pip3

```
apt-get install -y python3-pip
或
https://pip.pypa.io/en/stable/installing
```

# 安装virtualenv

```
pip3 install virtualenv
```

安装上述库后就可以使用virtualenv创建任意的python环境了。赶快动手试试把！！