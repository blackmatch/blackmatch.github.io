---
title: " 或许你真的不需要 moment.js"
description: 
date: 2018-12-17T21:49:43+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
---

## 前言

[momentjs](https://github.com/moment/moment/) 是一个很流行的处理日期、时间的库，这个库提供了很多友好的方法，方便开发者对日期时间进行比较、格式化等操作。然而，由于这个库是基于面向对象的思维逻辑设计的，每一个 moment 对象会自带很多属性和方法，在进行相关的操作时会进行很多校验、转换等操作，所以在对性能比较敏感的程序中可能会存在一定的问题，特别是在循环语句中，性能问题可能更加突出。

我在实际的项目中就遇到过在循环语句中使用 `momentjs` 导致性能问题的情景：从一个数组中取出符合条件的元素，条件之一是需要对元素的某个日期时间属性作判断，使用到了 `momentjs` 的 `.isAfter` 方法，数组中有 4 百多个元素，每次遍历一个元素需要 80ms，所以完成整个遍历需要 30 多秒，虽然使用场景不是提供 API 接口，但这也是很难接受的。后来换成了 `JavaScript` 原生的方法，每遍历一个元素只需要不到 2ms，遍历所有的元素不到一秒就完成了，性能瞬间就提升了。说明：我仅仅是修改了日期时间比较的代码，其他没有做任何优化。

## 简单测试

我们可以简单的测试一下 `momentjs` 和原生的 `JavaScript` 方法之间的性能差异，测试代码如下：

```js
const debug = require('debug')('moment-demo');
const moment = require('moment');

const items = [];
for (let i = 0; i < 500; i += 1) {
  const ts = new Date().getTime();
  const n1 = Math.floor(Math.random() * 1000);
  const n2 = Math.floor(Math.random() * 1000);

  items.push({
    n1,
    n2,
    time1: new Date(ts - n1 * 24 * 60 * 60 * 60),
    time2: new Date(ts - n2 * 24 * 60 * 60 * 60)
  });
}

debug(`items len: ${items.length}`);

const arr1 = [];
for (let i = 0, iLen = items.length; i < iLen; i += 1) {
  const item = items[i];
  if (moment(item.time2).isAfter(item.time1)) {
    arr1.push(item);
  }
}

debug(`arr1 len: ${arr1.length}`);

const arr2 = [];
for (let i = 0, iLen = items.length; i < iLen; i += 1) {
  const item = items[i];
  const t1 = new Date(item.time1).getTime();
  const t2 = new Date(item.time2).getTime();
  if (t2 > t1) {
    arr2.push(item);
  }
}

debug(`arr2 len: ${arr2.length}`);

```

测试结果如下：

![debug]({{ site.url }}/images/you-dont-need-momentjs/debug.jpg)

> 从结果可以看出，`momentjs` 在性能上确实逊色于原生的 `JavaScript` 方法。

## 相关库的比较

其实， github 上已经有一个项目专门分析 `momentjs` 与其他日期时间处理库的各项对比，项目地址在 [ 这里 ](https://github.com/you-dont-need/You-Dont-Need-Momentjs)。一个简要对比如图：

![vs]({{ site.url }}/images/you-dont-need-momentjs/vs.jpg)

## 我的建议

如果你只是用 `momentjs` 中的两三个方法，则完全可以考虑使用其他更加轻量、性能更加好的库，或者使用 `JavaScript` 原生的方法，必要时封装一下即可。如果程序对性能有较高的要求，最好不要使用 `momentjs`。