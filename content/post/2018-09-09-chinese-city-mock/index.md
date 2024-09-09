---
title: " 写了个省市县的 mock 模块 "
description: 
date: 2018-09-09T21:15:48+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
---

## 前言

现在的很多软件可能都会有省市县三级联动菜单，网上也有很多现成的数据。但是，我们在使用的时候往往会有一些顾虑：这是不是标准的数据？这数据是不是最新的啊？这数据不是我想要的格式。。。在进行开发的过程中有时候也需要造一些省市县相关的数据，大家第一反应可能会是 mock，但是我找了一下没有发现相关的模块，但是找到了一个 [mockjs](https://github.com/nuysoft/Mock/tree/refactoring)，于是我就依葫芦画瓢写了一个 [mockz](https://github.com/blackmatch/mockz) 模块。

## 科普

我国的行政区域划分其实是分为 5 个部分的，分别是省、市、县、镇、乡。通常软件设计的时候，只会让用户选择省、市、县三级，也就是我们常说的省市县三级联动菜单。制定这些标准的是 **国家统计局**，可以在 [ 这里 ](http://www.stats.gov.cn/tjsj/tjbz/tjyqhdmhcxhfdm/) 查看我国最新的行政区域划分标准。

## 方案设计

基于上面的说明，其实要做的就只有两件事：

* 从国家统计局获取最新的行政区域划分标准数据。
* 提供 mock 接口。

## 获取数据

通过爬虫获取最新的数据，这里不再赘述。爬取到数据后，将数据保存成 `.json` 文件，方便后续引用。数据的格式如下：

```javascript
[
  {
    "name": " 北京市 ",
    children: [
      {
        "code": "110100000000",
        "name": " 市辖区 ",
        children: [
          {
            "code": "110101000000",
            "name": " 东城区 "
          }
        ]
      }
    ]
  }
]
```

> 爬虫相关的代码暂时不公布，写得不好。使用爬虫的好处是：可以随时获取最新的数据，可以定制自己喜欢的数据格式。

## 提供 API 接口

```javascript
const mockz = require('mockz');

// 全部数据
const all = mockz.all();
console.log(all[0]);

// 随机获取一个省级单位
const province = mockz.province();
console.log(province);  // 山西省

// 随机获取一个市级单位
const city = mockz.city();
console.log(city);  // 咸阳市

// 随机获取某个省的一个市级单位
const city2 = mockz.city(' 广东省 ');
console.log(city2);  // 潮州市

// 随机获取一个县级单位
const county = mockz.county();
console.log(county);  // 怀柔区

// 随机获取某个省的一个县级单位
const county2 = mockz.county(' 广东省 ');
console.log(county2);  // 蕉岭县

// 随机获取某个省某个市的一个县级单位
const county3 = mockz.county(' 广东省 ', ' 广州市 ');
console.log(county3);  // 天河区

// 随机获取一个地址
const address = mockz.address();
console.log(address); // 山西省晋中市昔阳县
```

> `mockz.all()` 可用来做省市县三级联动菜单的数据源。

## 最后

每一件小事都值得用心思考，然后付诸行动，不求一鸣惊人，只愿每日精进。模块的源码地址如下：

[https://github.com/blackmatch/mockz](https://github.com/blackmatch/mockz)
