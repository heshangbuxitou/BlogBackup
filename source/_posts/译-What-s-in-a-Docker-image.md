---
title: '[译] What''s in a Docker image?'
date: 2018-12-12 15:08:05
tags:
- Docker
categories:
- 译文
---

原文链接：[https://cameronlonsdale.com/2018/11/26/whats-in-a-docker-image](https://cameronlonsdale.com/2018/11/26/whats-in-a-docker-image/)

平时用 `Docker` 的都知道 `Image` 是构成 `Container` 的二进制文件，至于里面的内容到底是什么并不知道，作者在这篇文章详细的分析 `Docke Image` 的组成内容和文件结构。

# Docker Image 中有什么？

这是一个非常好的问题，在你知道答案之前，`Docker Images` 看起来非常神秘。我不但要告诉你答案，还要告诉你我是怎么样知道答案的。

# 从Dockerfile 到Image

让我们从头开始讲起。希望你对 `Dockerfile` 非常熟悉 - 下面为你说明 `Docker` 怎样构建 `Image`  。这儿有一个简单的例子。

```shell
FROM ubuntu:15.04
COPY app.py /app
CMD python /app/app.py
```

上面的每一行是关于 `Docker` 构建 `Image` 的说明。它使用 `ubuntu:15.04` 系统作为基础，然后把 `app.py` 脚本复制到`ubuntu`的目录中