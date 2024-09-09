---
title: " 利用 Stream 批量处理文件 "
description: 
date: 2018-09-08T21:11:46+08:00
image: 
math: 
license: 
hidden: false
comments: true
categories:
  - 后端
tags:
  - Node.js
draft: false
---

## 应用场景

有 500 多个 `.txt` 文件，每个文件几十 M，需要把这些文件的内容全部写入到数据库中，文件中每一行是一条数据，每个文件大约 200 万条数据。

## 处理流程

![process](process.jpg)

这也是数据流动的过程，期望达到的效果是：**快速**、**高性能（占用内存小）**。这 3 个步骤中，`解析、处理文件内容` 这个步骤基本不会存在什么问题。问题的难点在 `读文件` 和 `写入数据库` 这两个步骤。

## 读文件

一看到读取大文件或者批量文件，可能大家第一反应就是 `stream`。我也是这样想的，但是这里不能直接使用可读流来读取文件，前面提到，每个文件中每一行是一条数据，但是通过可读流读取文件的时候是以 `buffer` 的形式（也可以设置其他形式，比如 `objectMode`）读取的，并不是按照一行一行的方式读取，或许通过一些技巧可以实现，但是没有必要，因为有现成的 `readline` 模块。`readline` 模块读取文件的思想和可读流是一样的，只不过该模块每一次读取的数据是文件中的一行而已，通过 `readline` 模块可以实现逐行读取文件，且不会造成内存压力。

## 写入数据

我使用的是 MySQL 数据库，因为文件中会有很多重复的内容，所以我给表的某个字段添加了 `UNIQUE` 索引，为了后期使用方便，我给 2 个字段添加了 `INDEX` 索引。这种做法对提高数据质量和查询速度有一定的帮助，但相对的，会对数据写入造成一定的影响，简单点说就是写入速度会变慢。本文不展开讨论如何提升数据库的写入速度，只是交代有这么个情况而已。写入数据库通过 [knex](https://knexjs.org/) 和 [mysql](https://github.com/mysqljs/mysql) 这两个模块来实现。

## 读取和写入速度差异的问题

我们姑且忽略中间解析、处理文件内容所消耗的时间，把注意力集中在数据的读取和写入上。上面提到，写入数据库的过程是比较慢的，此外每次有数据需要写入的时候都得先连接数据库，等待数据库反馈，然后再写入下一条数据。由于读取数据的速度是非常快的，一个文件 2 秒左右就读取完成了，但是要把一个文件的所有数据完全写入到数据库中需要花费 30 分钟左右，这种巨大的速度差异最终会给内存带来很大的压力，没有写入到数据库的数据一直在内存中堆积，直到最终内存爆了，程序异常退出。

## 解决读写速度差异问题

其实这个问题概括一下就是：**读太快了，写太慢了**。解决思路也就有 2 条了： 1. 读慢一点； 2. 写快一点。最理想的情况是读写速度趋向于平衡，且读写速度都非常快，但是要达到这种状态，以我目前的能力是不可能的（求大佬带）。所以我选择了第一种方案：**降低读取速度**。经过测试，完全读取一个文件的所有内容到内存中等待写入不会对我的机器造成内存压力，占用大概 100 多 M 内存。所以，我最终的解决方案是：**一次读取一个文件的数据，把这些数据全部扔给可写流，可写流把这些数据一条一条写入到数据库中，等可写流把队列中所有的数据都写入到数据库后，再读取下一个文件的内容，然后把数据继续扔到可写流中，如此循环操作，直到所有的文件都处理完成**。这样做会有 2 个好处：**1.不会造成内存压力；2.可写流中一直有数据在等待写入数据库，不会造成资源浪费**。

## 代码实现

```js
const dealFile = (file, writer) => {
  const dst = path.join('mail', file);

  const rl = readline.createInterface({
    input: fs.createReadStream(dst),
  });

  rl.on('line', (line) => {
    writer.write(line);
  });

  rl.on('close', () => {
    debug(`文件 ${file} 读取完毕`);
  });
};

class WStream extends Stream.Writable {
  constructor(opts) {
    super();

    this.counter = 0;
    this.txtFiles = opts.txtFiles;
    this.fileIdx = 0;

    this.on('nextFile', (file) => {
      dealFile(file, this);
    });
  }

  start() {
    this.emit('nextFile', this.txtFiles[this.fileIdx]);
  }

  _write(chunk, encoding, next) {
    const rx = new RegExp(/\S+[a-z0-9]@[a-z0-9\.]+/img);
    const mArr = chunk.toString().trim().match(rx);

    if (mArr && mArr.length > 0) {
      const email = mArr[0].trim();
      const password = chunk.toString().trim().replace(rx, '').replace('----', '').trim();

      if (email.length > 0 && password.length > 0) {
        knex('netease').insert({
          email,
          password,
        }).then((result) => {
          this.counter += 1;
          if (this.counter % 10000 === 0) {
            debug(`队列中还有 ${this.writableLength} 数据需要处理`);
            this.counter = 0;
          }
          next();
          if (this.writableLength === 0) {
            this.fileIdx += 1;
            if (this.fileIdx < this.txtFiles.length) {
              this.emit('nextFile', this.txtFiles[this.fileIdx]);
            } else {
              debug('all done!');
              process.exit(0);
            }
          }
        }).catch((err) => {
          this.counter += 1;
          if (this.counter % 10000 === 0) {
            debug(`队列中还有 ${this.writableLength} 数据需要处理`);
            this.counter = 0;
          }
          next();
          if (this.writableLength === 0) {
            this.fileIdx += 1;
            if (this.fileIdx < this.txtFiles.length) {
              this.emit('nextFile', this.txtFiles[this.fileIdx]);
            } else {
              debug('all done!');
              process.exit(0);
            }
          }
        });
      } else {
        next();
      }
    } else {
      next();
    }
  }
}

const run = () => {
  const files = fs.readdirSync('./mail');
  const txtFiles = _.filter(files, (file) => {
    return file.endsWith('.txt');
  });

  debug(`共有 ${txtFiles.length} 个文件`);

  const writer = new WStream({
    txtFiles,
  });

  writer.start();
};

run();
```

上述是核心代码，写得有点烂，还有很多可以优化的空间，这里我们只看思路就好了。几个关键点：

* 写一个可写流的类，继承 `Stream.Writable`，并实现 `_write` 方法。
* 每读取文件中的一行数据，就调用可写流的 `.write` 方法，把这条数据扔到可写流中等待处理。
* 判断可写流中有没有数据需要处理，通过 `writable.writableLength` 属性来判断，这个属性是在 `Node.js` 的 `v9.4.0` 版本中加入的。需要注意的是，这个属性在这里指的是字节数，不是行数。
* 需要在调用 `next()` 方法之后再去判断 `writable.writableLength` 的值，否则会出现程序卡死，因为如果在 `next()` 方法之前判断的话，处理可写流队列中最后一条数据的时候 `writable.writableLength` 的值是大于 0 的，而这又是最后一次调用可写流的 `.write` 方法，所以不会继续读取下一个文件了，造成程序卡死。

## 总结一下

`Node.js` 的 `stream` 模块及其理念在原生模块中应用非常广泛，比如：`HTTP.Request`、`HTTP.response`、`process.stdin`、`process.stdout` 等。这其实类似于 **大事化小小事化了** 的思想，我一开始也尝试过一些其他的解决方案，比如：

* 并发读取多个文件，通过 `Promise` 和 `async/await` 控制并发。
* 一个文件一个文件读取，读取完一个文件后等待 25 分钟再读取下一个文件。

这两种方案都不是很理想，没能解决 **读写速度差异问题**。由于 `stream` 用得也不多，所以走了一些弯路，但就结果而言，目前的解决方案能够比较优雅的解决了我的问题，同时也能学到了一些东西。文章肯定存在很多纰漏，欢迎各位大佬指正。