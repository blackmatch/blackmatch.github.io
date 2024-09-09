---
title: " 使用 Node.js 进行高并发请求时需要注意 DNS 解析问题 "
description: 
date: 2018-10-13T21:47:00+08:00
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

我们可以使用 Node.js 的 `http` 模块进行网络请求，比如使用 `http.get` 方法进行 __GET__ 请求。当时在高并发请求的情况下，很容易出现如下的错误：

```plain
Error: getaddrinfo ENOTFOUND www.baidu.com www.baidu.com:80
    at GetAddrInfoReqWrap.onlookup [as oncomplete] (dns.js:57:26)
```

或者是这样的错误：

```plain
Error: queryA ETIMEOUT www.baidu.com
    at QueryReqWrap.onresolve [as oncomplete] (dns.js:197:19)
```

这两个错误都和 `DNS` 解析有关。

## 一种解决方案

在 Google 搜索一番就会找到如下的解决方案：

```javascript
const opts = {
  host: 'www.baidu.com',
  family: 4
};

http.get(opts, (res) => {
  // handle the response
})
```

关键点是在请求的时候添加了 `family` 这个参数，我们先来看一下这个参数的官方解释：

> family <number> IP address family to use when resolving host and hostname. Valid values are 4 or 6. When unspecified, both IP v4 and v6 will be used.

大概的意思是：这个参数可以指定解析 `host` 和 `hostname` 的时候所使用的 IP 地址族。可接受的参数为 `4` 和 `6` 。如果不指定，会同时使用 IP-v4 和 IP-v6 。

所以，这种解决方案就是让 Node 在调用 `dns` 模块解析域名的时候，指定使用 IP-v4 。这样能够在一定程度上解决问题。为什么说们说是一定程度上呢？因为经过我的验证，当请求并发量继续增大的时，还是会存在问题。

## 一些猜想

Node 使用 `dns` 模块来提供域名解析服务，当我们调用 `http` 、 `net` 等相关模块时，也会使用到 `dns` 模块来进行相关的操作。当然，`dns` 模块也可以单独使用，例如：

```javascript
const dns = require('dns');

const len = 10000;

const run = () => {
  for (let i = 0; i < len; i++) {
    dns.resolve4('www.baidu.com', (err, addresses) => {
      if (err) {
        console.log(`i: ${i}`);
        throw err;
        process.exit(0);
      }
      console.log(addresses[0]);
    });
  }
};

run();

```

那为什么高并发请求的时候会出现问题呢？我的猜想是：`dns` 模块对域名解析有一定的性能限制，当并发量达到一定程度时，就会出现超时，从而导致各种问题。那为什么使用 IP-v4 就能得到一定程度的改善呢？我的猜想是：默认情况下，`dns` 模块使用的是 IP-v4 和 IP-v6 进行域名解析，在切换解析规则时或者使用不同的规则对性能有一定的依赖，当指定使用 IP-v4 的时候，能够使得 `dns` 模块发挥最佳的性能，从而使问题得到一定的改善。

## 一些建议

在一些高并发请求的场景（比如爬虫）下，很有可能会导致 DNS 无法正常解析的问题，比如使用 `request` 模块进行高并发请求的时候也会出现问题。除了上述提到的解决方案外，还应该合理控制好并发量。此外，也可以尝试使用高性能的 DNS 服务器来提供 DNS 解析的效率。

## 参考资料

[http://www.ruanyifeng.com/blog/2016/06/dns.html](http://www.ruanyifeng.com/blog/2016/06/dns.html)

[https://github.com/nodejs/node/issues/1644](https://github.com/nodejs/node/issues/1644)
