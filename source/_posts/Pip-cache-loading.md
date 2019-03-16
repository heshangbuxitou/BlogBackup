---
title: pip缓存加载
date: 2017-11-12 00:34:18
tags:
- python
categories: 
- 技术杂谈
---

Python最常用的的包管理工具就是pip了，无论是安装包还是卸载包都非常好用，但是在我使用pip的时候出现了一些因为pip默认安装产生的bug。

当时是部署项目到云服务器，安装好uwsgi运行的时候出现`no internal routing support, rebuild with pcre support`，后来发现是pcre库没有安装的原因，当我重新安装uwsgi时， pip 好像是直接从缓存中拿出了上次编译后的 uwsgi 文件，并没有重新编译一份。于是翻了一下pip的手册说明带上`--no-cache-dir`可以强制下载重新编译安装的库，问题解决了。

pip下载缓存的默认位置到底在哪里呢？

在pip源码的`user_cache_dir`中说明了，缓存目录一般在当前用户`home`目录下的`.cache`文件夹中，当你把里面的`pip`文件夹删除后，重新安装uwsgi库的时候就会重新编译了。

另外介绍一个小技巧，当你使用pip下载安装python库的时候可以使用豆瓣的下载源，技巧是在`pip install <Library>`后面加上`--index-url=http://pypi.douban.com/simple --trusted-host=pypi.douban.com`这么一段代码。相信我，用了之后你就换不了了。嘿嘿！
