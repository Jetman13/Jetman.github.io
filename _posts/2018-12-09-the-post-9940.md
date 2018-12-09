---
categories: Uncategoried
tags: 
layout: post
published: false
title: Unnamed
---
---
layout:     post
title:      "mybatis cache 源码解析"
subtitle:   " \"mybatis cache\""
date:       2018-12-09 12:00:00
author:     "Jetman"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 源码
    - mybatis
    - 技术
---



## 前言

mybatis的cache部分是做db操作的缓存特性。分为一级缓存和二级缓存，虽然工程里面一般用redis做db的分布式缓存，但是mybatis也有提供这种单机的缓存特性。下面通过分析一级和二级缓存特性和源码。最后总结mybatis缓存源码对工程代码的思考。

##一级缓存
一级缓存是在同个数据库会话里面同事查询相同的sql，mybatis可以对第一次的db查询做缓存，保证第二次相同的查询不会穿透db。
首先看一级缓存的工作模式

##二级缓存

###设计模式

##引发对工程代码的思考

