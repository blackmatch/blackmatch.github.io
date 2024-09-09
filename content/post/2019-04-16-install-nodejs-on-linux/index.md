---
title: " 在 Linux 系统上通过二进制包安装 Node.js"
description: 
date: 2019-04-16T21:57:25+08:00
image: 
math: 
license: 
hidden: false
comments: true
categories:
  - DevOps
tags:
  - Node.js
draft: false
---

## 前言

Linux 有很多个发行版本，不同的发行版本有不同的包管理工具。为了安装指定的 Node.js 版本，有时候需要花一些精力找攻略或者安装额外的包管理工具等，有些包管理工具并没有最新的 Node.js 版本。所以，如果是 Linux 系统，索性直接使用编译好的二进制文件进行安装是最省心省力的。

## 安装

* 下载指定版本的二进制文件

在 Node.js 官方的发布网站 [https://nodejs.org/dist/](https://nodejs.org/dist/) 下载合适的二进制包，比如我要安装 `v11.14.0` 版本，我需要下载二进制包为 [node-v11.14.0-linux-x64.tar.gz](https://nodejs.org/dist/v11.14.0/node-v11.14.0-linux-x64.tar.gz)。

* 解压文件

```shell
tar -xvf node-v11.14.0-linux-x64.tar.gz
```

* 拷贝文件到指定目录

```shell
sudo cp -r node-v11.14.0-linux-x64/* /usr/local/
```

* 测试是否安装成功

```shell
root@blackmatch:~# node -v
v11.14.0
root@blackmatch:~# npm -v
6.7.0
root@blackmatch:~# npx -v
6.7.0
```

## 总结

* 二进制包一定要下载合适的（比如 x64 、 x86 ）等。
* 安装完成后如果相关命令不生效，请重新打开一个终端即可生效。