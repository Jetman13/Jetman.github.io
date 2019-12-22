---
layout:     post
title:      "mysql隔离性"
subtitle:   " \"mysql隔离性\""
date:       2019-12-22 14:10:00
author:     "Jetman"
header-img: "img/in-post/mysql/mysql-5.jpg"
catalog: true
tags:
    - mysql
    - 技术
    - 总结
---


####  事务隔离级别

ACID（Atomicity、Consistency、Isolation、Durability，即原子性、一致性、隔离性、持久性）
##### 隔离性
1. 读未提交
> 两个事务，第一个事务读到另外一个事务未提交的修改。
2. 读已提交
> 两个事务，第一个事务需要等到第二个事务提交后才能查询到修改。
3. 可重复读
> 只要在一个事务中，无论这条记录是否被其他事务更改，在这个事务里面重复读都一样。
4. 串行化
> 对于同一条记录，读会加“读锁”，写会加“写锁”当出现读写冲突是，必须串行处理。

##### 视图
隔离性主要利用一致性读视图。
>    一致性读视图是利用事务id递增作为版本号和undolog做到可见性。
>    例如一个视图的事务id=3，查询数据库里面数据事务id=5，则5的数据对于视图来说不可见，  只能通过undolog回滚。一直回滚到id=3，代表该值才是事务可见性的值。
>    用此方式来实现读已提交和可重复读两种隔离性。

``` sql 
#查询当前超过60s的长事务
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```

#### 事务的隔离
1. innodb支持RC和RR隔离级别实现是用的一致性视图(consistent read view)

2. 事务在启动时会拍一个快照,这个快照是基于整个库的.
基于整个库的意思就是说一个事务内,整个库的修改对于该事务都是不可见的(对于快照读的情况)
如果在事务内select t表,另外的事务执行了DDL t表,根据发生时间,要嘛锁住要嘛报错

3. 事务是如何实现的MVCC呢?
- a. 每个事务都有一个事务ID,叫做transaction id(严格递增)
- b. 事务在启动时,找到已提交的最大事务ID记为up_limit_id。
- c. 事务在更新一条语句时,比如id=1改为了id=2.会把id=1和该行之前的row trx_id写到undo log里,
并且在数据页上把id的值改为2,并且把修改这条语句的transaction id记在该行行头
- d. 再定一个规矩,一个事务要查看一条数据时,必须先用该事务的up_limit_id与该行的transaction id做比对,如果up_limit_id>=transaction id,那么可以看.如果up_limit_id<transaction id,则只能去undo log里去取。去undo log查找数据的时候,也需要做比对,必须up_limit_id>transaction id,才返回数据

4. 什么是当前读,由于当前读都是先读后写,只能读当前的值,所以为当前读.会更新事务内的up_limit_id为该事务的transaction id

5. 为什么rr能实现可重复读而rc不能,分两种情况
(1)快照读的情况下,rr不能更新事务内的up_limit_id,
而rc每次会把up_limit_id更新为快照读之前最新已提交事务的transaction id,则rc不能可重复读
(2)当前读的情况下,rr是利用record lock+gap lock来实现的,而rc没有gap,所以rc不能可重复读