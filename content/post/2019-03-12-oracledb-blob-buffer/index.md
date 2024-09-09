---
title: " 记一次从 Oracle 数据库取 BLOB 数据遇到的坑 "
description: 
date: 2019-03-12T21:54:03+08:00
image: 
math: 
license: 
hidden: false
comments: true
categories:
  - 后端
tags:
  - 数据库
draft: false
---

## 需求

![requirement]({{ site.url }}/images/oracle-blob-buffer/requirement.jpg)

源库中的数据是以 `BLOB` 的形式存储的，且数据中含有中文， MySQL 数据库的字符集为 `utf8`，最终想要的效果就是在浏览器中以文本的形式展示源库中的数据。为了实现这一需求，尝试了 2 种方案：

* 从 Oracle 层面解决，通过视图将相关字段转换成 `VARCHAR2` 类型后在返回，这样从 Oracle 中查询数据的时候，直接拿到的就是字符串类型的数据。这样做的弊端是： Oracle 数据库 VARCHAR2 类型最大只能支持 4kb，如果超过了这个大小就会出错。
* 从 Oracle 取到数据后，使用 Node.js 转换成字符串后再存入到 MySQL 数据库中。

我使用了第 2 种解决方案，但是过程并不是很顺利。

## 遇到的问题

从 Oracle 数据库中取到的数据，在 Node.js 中是 `Buffer` 对象，要将 Buffer 对象转换成字符串对 Node.js 来说实在是太常规了，直接 `buffer.toString` 就完事了，可事实并非如此，得到的字符串都是乱码。一般遇到这个问题，大家的第一反应肯定是编码问题，我也是这么想的，考虑到数据中有中文，而 Node.js 原生并没有支持中文的相关编码，默认是 `utf8`，已经尝试过了。所以就引入了 [iconv-lite](https://github.com/ashtuchkin/iconv-lite) 这个模块，用来对 Buffer 对象进行解码，但是 Oracle 中使用的字符集是 `SIMPLIFIED CHINESE_CHINA.ZHS32GB18030`，所以我想当然的就使用 `GB18030` 编码来解码，代码示例：

```js
const iconv = require('iconv-lite');

// Convert from an encoded buffer to js string.
const str = iconv.decode(buffer, 'gb18030');
```

结果得到的字符串还是乱码，然后我又把 iconv-lite 支持的所有中文编码又试了一遍，得到的字符串全都是乱码。

## 解决

经过一番 Google 和尝试后仍然没有解决，然后就在上述提到的两种方案之间来回折腾。后来在朋友的引导下，得到了一个思路：先探测 Buffer 对象的编码，得到确定的编码后，再进行解码。于是乎就找到了这个模块：[detect-character-encoding](https://github.com/sonicdoe/detect-character-encoding)。这个模块主要是用来探测字符编码的，使用方法也很简单，示例代码：

```js
const fs = require('fs');
const detectCharacterEncoding = require('detect-character-encoding');

const fileBuffer = fs.readFileSync('file.txt');
const charsetMatch = detectCharacterEncoding(fileBuffer);

console.log(charsetMatch);
// {
//   encoding: 'UTF-8',
//   confidence: 60
// }
```

于是乎就用这个模块对上述提到的 Buffer 对象进行探测，得到的编码竟然是 `UTF-16LE`，然后使用这个编码进行解码，果然得到了正确的字符串。问题到此彻底解决了。

## 注意事项

* 探测编码时请多用一些数据样例来探测，最后使用可信度最高的编码。
* 千万不要动态探测编码，然后动态解码，因为这个模块的探测结果是随着数据的变化而变化的。
* 使用 iconv-lite 模块解码时，如果编码名称中有字母，请一律使用小写字母。
* 一定要确保从 Oracle 取到的数据在 Node.js 环境中为 Buffer 对象。

## 其他说明

* 连接 Oracle 使用的模块是 [oracledb](https://github.com/oracle/node-oracledb)
* 连接 MySQL 使用的模块是 [knex](https://github.com/tgriesser/knex)

## 总结

这次遇到的问题，其实解决方案是比较清晰的，但是在对 Buffer 进行解码遇到问题后没有冷静下来分析，在 2 个解决方案之间来回折腾浪费了很多时间；当已经很明确问题出现在哪个环节时，应该借助相关工具进一步确认问题的根源所在，比如：这次在解码环节出现了问题，而问题的根源也比较清晰，就是解码时使用的编码不对，所以就应该先明确 Buffer 对象所使用的编码，然后再用正确的编码进行解码即可。