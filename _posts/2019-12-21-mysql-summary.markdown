---
layout:     post
title:      "mysql总体设计图"
subtitle:   " \"mysql设计图\""
date:       2019-12-21 12:00:00
author:     "Jetman"
header-img: "img/in-post/bg-vim.jpg"
catalog: true
mermaid: true
tags:
    - mysql
    - 技术
    - 总结
---


## 前言

  关系型数据库是我们后端开发时常接触的和使用的中间件工具,但是除了sql语法外我们还需要关注其他原理,例如索引,锁,分布式同步等,下面以mysql为例子给出mysql的设计流程图
  
  
### 1. mysql执行流程图

![img](/img/in-post/mysql/mysql-summary.png)

![img](/img/in-post/mysql/mysql-flow.png)

>客户端：jdbc，mysql -u -p，等等
 
>服务端：连接器，查询缓存（mysql 8.0后已经没了），分析器，优化器，执行器
>负责从客户端的sql里面提取，分析，简历执行计划

>存储引擎：innodb，MyISAM，Memory等存储引擎，不同存储引擎数据的存储结构不一样，主要对执行器暴露执行接口（查询更新等）