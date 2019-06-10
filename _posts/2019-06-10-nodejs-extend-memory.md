---
layout: post
title: "Node.js中扩展内存那些事"
---

## 前言

Node.js使用的是Google的V8作为JavaScript脚本引擎，由于V8引擎的限制，Node.js中只能使用部分内存：**64位操作系统下约为1.4G，32位操作系统下约为0.7G**。对于浏览器来说，这样的限制影响不大，但是对于服务端程序来说有时候可能就不能满足需求了。虽然大内存的使用场景也相对较少，但还是会存在一些默认情况下非代码因素造成的内存溢出问题。

## 通用的解决方案

在网上搜到的大部分答案都是使用`--max_old_space_size`这个flag来扩展Node.js可使用的最大的老生代的内存。比如：

```shell
node --max_old_space_size=2048 xxx.js
```

> 单位是MB，比如上述例子是将最大可用内存扩展至2G。

如果你使用的是Node8.x及以上的版本，还可以通过`NODE_OPTIONS`这个系统环境变量来配置，例如：

```shell
export NODE_OPTIONS=--max_old_space_size=2048
```

必须要确认该系统环境变量已经生效了，可以用`echo $NODE_OPTIONS`来校验。

此外，**使用`Buffer`不受V8内存限制**。

## 验证扩展内存是否生效

通过`v8`模块的`getHeapStatistics()`方法即可。

```js
const v8 = require('v8')

console.log(v8.getHeapStatistics())
```

执行`node --max_old_space_size=2048 test.js`结果如下：

```js
{
  total_heap_size: 5009408,
  total_heap_size_executable: 1048576,
  total_physical_size: 3449648,
  total_available_size: 2194521344,
  used_heap_size: 2411032,
  heap_size_limit: 2197815296,
  malloced_memory: 8192,
  peak_malloced_memory: 410288,
  does_zap_garbage: 0
}
```

其中的`heap_size_limit`就是老生代可以使用的最大内存，单位是`Byte`。

> 注意：Node8.x以下版本使用这个方法有bug：当设置的内存超过3.7G时，打印出来的结果是有问题的。

## 一些特殊情况

在实际的项目中，一般不会很简单的执行`node xxx.js`来启动项目，有些可能通过`pm2`等工具来部署，有些可能通过自己写的npm命令行工具来启动，而恰好这个时候使用的Node.js版本是低于8.x版本，这种情况下就只剩下添加`--max_old_space_size`这个flag来扩展内存了，但是怎么加还是有讲究的，比如一个npm命令行模块的入口文件的第一行通常是：

```shell
#!/usr/bin/env node
```

这样子的，所以我们很容易就想到：

```shell
#!/usr/bin/env node --max_old_space_size=2048
```

通过这样的方式来扩展内存，但事实上这种方式是有很大的局限性，甚至还有可能导致程序卡死。比如如果源代码这样写：

```js
#!/usr/bin/env node --max_old_space_size=2048

const program = require('commander')

program
  .version('0.1.0')
  .command('start')
  .action(function () {
    // start process...
  })

program.parse(process.argv)

...
```

假设我们这个包名叫`blog`，在服务器上全局安装这个包后执行`blog start`命令就能将程序启动起来。

但是，事与愿违，其实在服务器上执行`blog start`的时候，真正执行的是`blog --max_old_space_size=2048`，`start`这个参数无法传递下去，因为已经被`--max_old_space_size=2048`这个flag占用了。这是`#!`写法的一个特性。

## 使用hack技巧解决特殊情况

针对上述的特殊情况，曾经有人向Node.js提过一个[PR](https://github.com/smikes/node/blob/minus-x-switch/doc/Minus-X-Switch-Proposal.md)，但是后来发现，不需要Node做任何修改，使用shell的hack技巧就能解决，具体如下：

```js
#!/bin/sh 
":" //# comment; exec /usr/bin/env node --max_old_space_size=2048 "$0" "$@"

const program = require('commander')

program
  .version('0.1.0')
  .command('start')
  .action(function () {
    // start process...
  })

program.parse(process.argv)

...
```

解释一下：

* 第一行是指定使用`sh`shell。
* 第二行其实分为两个部分：`":" //# comment;` 和 `exec /usr/bin/env node --max_old_space_size=2048 "$0" "$@"`。前半部分是[Bourne Shell](https://www.gnu.org/software/bash/manual/html_node/Bourne-Shell-Builtins.html#Bourne-Shell-Builtins)内置支持的写法，只需要了解`#`和`;`之间是可以写注释的就行了；后半部分是使用Node执行相关的命令，`$0`传递一个参数0，`$@`传递其他额外的参数。

这样，就能正常传递参数了。

## 参考资料

http://sambal.org/2014/02/passing-options-node-shebang-line/