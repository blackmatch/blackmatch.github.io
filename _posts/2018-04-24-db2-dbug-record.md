---
layout: post
title: "记一次使用db2数据库遇到的坑"
---

IBM DB2 是美国IBM公司开发的一套关系型数据库管理系统，它主要的运行环境为UNIX（包括IBM自家的AIX）、Linux、IBM i（旧称OS/400）、z/OS，以及Windows服务器版本。 DB2主要应用于大型应用系统，具有较好的可伸缩性，可支持从大型机到单用户环境，应用于所有常见的服务器操作系统平台下。

我需要实现的技术方案如下：

![debug]({{ site.url }}/images/db2-debug-record/structure.webp)

需要从db2数据库中取数，然后把数据写入到MySQL数据中。

## 理论知识

DB2数据库几个概念

instance, 同一台机器上可以安装多个DB2 instance。

database, 同一个instance下面可以创建有多个database。

schema, 同一个database下面可以配置多个schema。 所有的数据库对象包括table、view、sequence，etc都必须属于某一个schema。

另外，database是一个connection的目标对象，也就是说用户发起一个DB2连接时，指的是连接到到一个database，而不是连接到一个instance，也不是连接到一个schema。

但是DB2的启动和关停是以instance为单位的。可以启动一个instance，或者关停一个instance。但不可以启动或者关停一个数据库或者一个schema。

## 使用的模块

使用了ibm_db，该模块在安装时会根据当前平台自动下载对应的客户端驱动程序。

第一坑
遇到的第一坑是CODEPAGE（代码页），可以简单的理解为这是数据库的编码，在db2数据库数据库中，如果客户端和服务端的CODEPAGE不一致，连接时会报错：

```plaintext
SQL0332N  Character conversion from the source code page "1386" to the target code page "819" is not supported
```

而使用的ibm-db中没有提到如何设置CODEPAGE的方式，在各种google攻略后，得到的解决方案有两个：

* 修改客户端的操作系统语言。

* 添加系统环境变量`DB2CODEPAGE`。

我的后端架构为：

![debug]({{ site.url }}/images/db2-debug-record/server-layer.webp)

最外层是ubuntu系统，然后起一个容器，容器是基于ubuntu16.04的，然后在容器中有一个ETL模块，这是一个node模块，node通过调用这个模块去连接db2数据库。

不论我怎么修改最外层的ubuntun系统还是容器中的unbuntu系统的语言和环境变量，都不起作用。

最终解决方案：

在`ETL module`模块中，在连接db2代码之前设置环境变量：

```plaintext
process.env.DB2CODEPAGE = 1386;
```

## 第二坑

成功连接到db2数据库后，发现取到的数据的中文是乱码。于是又开始在网上找攻略，大多数答案都是说`CODEPAGE`问题。可是上一个坑已经解决了呀。优于无法直接访问到db2所在的服务器，所以无法很准确的确认db2数据库使用的`CODEPAGE`值，但是经过各种调试及执行如下SQL语句：

```plaintext
SELECT CODEPAGE FROM SYSCAT.DATATYPES WHERE TYPENAME = 'VARCHAR';
```

得出的结论都是：db2数据库的`CODEPAGE`是1386，可以理解成db2数据库的编码是GBK。所以我把客户端的CODEPAGE设置成1386应该是没有问题的呀？但是实际情况就是中文无法正确展示。

最终解决方案：

```plaintext
process.env.DB2CODEPAGE = 1208;
```

将客户端的CODEPAGE设置成1208即可。一脸懵逼啊！此方案是我拍脑袋尝试后得出的。1208对应的编码是utf8。

## 其他坑

* 环境问题，项目自身的打包发布流程存在各种坑。

* ibm_db问题，2.1.0之前的版本在连接上是存在一些问题的，我开始折腾的时候是2.3.0版本，其实这个版本也有一些问题，然后我提了个issue，过了两天后更新到了2.3.1，使用这个版本后神奇的解决了连接问题。

* 使用ibm_db去连接db2数据库，不是真正的命令行客户端连接，而是使用了一个驱动程序去连接，所以网上的在客户端执行`db2set DB2CODEPAGE=1386`的方法都行不通。

* 使用Node.js连接db2数据库的相关资料较少，使用的人也少，搜到很多都是java的资料。

## 总结

为了实现开头提到的技术方案，我加班加点花了差不多一个礼拜的时间，除了项目本身的打包、运行环境的坑以及对db2不熟悉外，其他问题大概花了两天左右。经过一个礼拜的折腾，对解决问题有了一些心得：

* 环境很重要，因为我在本地开发环境是执行代码是比较顺利的，但是现场环境比较复杂，所以在解决问题之前要充分了解现场的软件环境，包括操作系统、版本等。

* 如果是没遇到的东西，最后先去了解基本的概念、必要的基础知识。

* 从最终代码执行处入手，如果我一开始就在使用代码连接db2的地方通过`process.env.DB2CODEPAGE`打印出来的话，可能会省很多时间。

## 参考资料

[https://www.jianshu.com/p/e1f38505f789](https://www.jianshu.com/p/e1f38505f789)
