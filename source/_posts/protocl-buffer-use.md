---
title: protocol buffer使用
date: 2017-11-30 11:03:14
tags:
---
# 引言

在我想使用`protobuf`来序列化数据时，发现我的系统是`windows`，网上有许多安装`protobuf`库的实例，我都试过，安装都失败了，然后我只能自己来了，凭着经验和猜想以及前人的安装经验，我终于是安装成功了，现在把安装记录下来，方便以后需要的人可以直接使用。

# win7 下配置:


1.下载对应的包 ，下载链接`https://github.com/os72/protoc-jar`，下载完之后把`protoc-2.5.0-windows-x86_64.exe`复制到对应的Python执行环境的`Scripts`目录下并改名为`protoc.exe`，这里有一个坑就是你的`protocol_buffer`要和`protoc`的版本对应，一开始我是用的2.5.0版本的，结果出现了`google/protobuf/descriptor.proto:428:3: Expected "required", "optional", or "repeated".`错误，然后我换了3.4.0版本的`protoc.exe`就成功了。


2.下载protobuf并解压，下载链接`https://github.com/google/protobuf`,通过cmd进入到对应目录下的Python目录中，执行`python setup.py build`,可以看到在对应的`google`目录中会生成对应的文件，然后你把对应的`google`目录拷贝到你的`python`的`Lib/site-packages`目录下就可以使用了。

# centos下的安装:

`yum -y install autoconf automake libtool curl make g++ unzip`
**注意在`g++`在centos中叫`gcc-c++`所以需要`yum -y install gcc-c++`。**

先在`https://github.com/google/protobuf`上下载你需要的protobuf版本，然后解压到一个目录，我下载的是2.5.0版本的protobuf，通过`tar -zxvf proto -C /usr/local/`放在`/usr/local/`目录下。

然后下载`googletest`库，国外下载有点慢，我这有一个国内的链接[]()（注意：这个库是需要的，不然后面的安装可能会报错，我前面在网上看的没这个步骤安装一直报错），把它解压放到
`/usr/local/your-protobuf-release/`目录下，改名为`gtest`。

然后依次执行
（1）`./autogen`
（2）`./configure`
（3）`make` 
（4）`make check`
（5）`make install`

```
cd python
python setup.py build
python setup.py test
python setup.py install
ls
```
