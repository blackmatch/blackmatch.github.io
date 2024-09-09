---
title: " 请不要使用 require 引入单个文件 "
description: 
date: 2019-06-11T22:04:28+08:00
image: 
math: 
license: 
hidden: false
comments: true
categories:
  - 后端
tags:
  -  Node.js
draft: false
---

## 前言

Node.js 的模块化是基于 CommonJS 规范实现的，我们通常会使用 `module.exports` 来导出一个模块，用 `require` 来引入一个模块。其实在 Node.js 中，一个文件就是一个模块，更多时候我们使用 `require` 来引入一些 NPM 包。例如：

```js
const _ = require('lodash')

// codes
```

但是有时候我们也需要引入一些文件，最常见的文件就是 `.json` 文件，例如：

```js
const package = require('./package.json')

// codes
```

用这种方式引入单个文件在大多数情况下是可行的，但是如果引入的文件是一个可能会在程序启动后发生变化的文件就会有问题了。

## require 的缓存机制

当程序启动后， Node.js 会在当前进程缓存所有用 require 引入过的内容，并保存在全局对象 `require.cache` 中。所以，如果使用 require 引入一个动态文件，在程序运行过程中就无法获取最新的文件内容了。

我们来看一个测试：

```js
const fs = require('fs')

const test = () => {
  // 首次引用 config.json
  const config1 = require('./config.json')
  console.log('first require config')
  console.log(config1)

  console.log('require cache:')
  console.log(require.cache)

  // 修改 config.json
  const str = fs.readFileSync('./config.json', 'utf8')
  const obj = JSON.parse(str)
  obj.age = 20
  fs.writeFileSync('config.json', JSON.stringify(obj, null, 2))

  // 第二次引用 config.json
  const config2 = require('./config.json')
  console.log('second require config')
  console.log(config2)
}

test()
```

`config.json` 文件中的内容为：

```json
{
  "name": "blackmatch",
  "age": 18
}
```

输出结果：

```txt
first require config
{ name: 'blackmatch', age: 18 }
require cache:
{ '/Users/blackmatch/Desktop/blackmatch/demos/require-cache-demo/test.js':
   Module {
     id: '.',
     exports: {},
     parent: null,
     filename: '/Users/blackmatch/Desktop/blackmatch/demos/require-cache-demo/test.js',
     loaded: false,
     children: [ [Object] ],
     paths:
      [ '/Users/blackmatch/Desktop/blackmatch/demos/require-cache-demo/node_modules',
        '/Users/blackmatch/Desktop/blackmatch/demos/node_modules',
        '/Users/blackmatch/Desktop/blackmatch/node_modules',
        '/Users/blackmatch/Desktop/node_modules',
        '/Users/blackmatch/node_modules',
        '/Users/node_modules',
        '/node_modules' ] },
  '/Users/blackmatch/Desktop/blackmatch/demos/require-cache-demo/config.json':
   Module {
     id: '/Users/blackmatch/Desktop/blackmatch/demos/require-cache-demo/config.json',
     exports: { name: 'blackmatch', age: 18 },
     parent:
      Module {
        id: '.',
        exports: {},
        parent: null,
        filename: '/Users/blackmatch/Desktop/blackmatch/demos/require-cache-demo/test.js',
        loaded: false,
        children: [Object],
        paths: [Object] },
     filename: '/Users/blackmatch/Desktop/blackmatch/demos/require-cache-demo/config.json',
     loaded: true,
     children: [],
     paths:
      [ '/Users/blackmatch/Desktop/blackmatch/demos/require-cache-demo/node_modules',
        '/Users/blackmatch/Desktop/blackmatch/demos/node_modules',
        '/Users/blackmatch/Desktop/blackmatch/node_modules',
        '/Users/blackmatch/Desktop/node_modules',
        '/Users/blackmatch/node_modules',
        '/Users/node_modules',
        '/node_modules' ] } }
second require config
{ name: 'blackmatch', age: 18 }
```

从输出结果可以看出：

* config.json 文件被缓存到了 require.cache 全局对象中了。
* 在 config.json 文件被修改后，第二次使用 require 引入文件无法获取该文件最新的内容。

这就是 require 的缓存机制。

## 使用读取文件的方式引入动态文件

同样基于上面的例子，我们使用读取文件的方式引入 config.json，代码如下：

```js
const fs = require('fs')

const test2 = () => {
  // 首次引用 config.json
  const str1 = fs.readFileSync('./config.json', 'utf8')
  const config1 = JSON.parse(str1)
  console.log('first require config')
  console.log(config1)

  // 修改 config.json
  const str = fs.readFileSync('./config.json', 'utf8')
  const obj = JSON.parse(str)
  obj.age = 20
  fs.writeFileSync('config.json', JSON.stringify(obj, null, 2))

  // 第二次引用 config.json
  const str2 = fs.readFileSync('./config.json', 'utf8')
  const config2 = JSON.parse(str2)
  console.log('second require config')
  console.log(config2)
}

test2()
```

输出结果：

```txt
first require config
{ name: 'blackmatch', age: 18 }
second require config
{ name: 'blackmatch', age: 20 }
```

这次就能获取 config.json 最新的内容了。

## 总结

* 引入一个动态文件的场景比较少，但养成使用读取文件的方式来引入单个文件是个好习惯。
* 由于 require 的缓存机制，在需要热更新的场景可能需要另辟蹊径，必要时重启进程。
