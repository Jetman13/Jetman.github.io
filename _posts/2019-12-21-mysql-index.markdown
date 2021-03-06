---
layout:     post
title:      "mysql索引"
subtitle:   " \"innodb索引\""
date:       2019-12-21 12:00:00
author:     "Jetman"
header-img: "img/in-post/mysql/mysql-3.jpg"
catalog: true
tags:
    - mysql
    - 技术
    - 总结
---


#### 索引
##### 索引常见模型:bTree,hash,有序数组
  - a. hash:数据结构参考jdk hashMap.
    优点:单个查找非常快.
    缺点:不支持区间查找,区间查找需要全表扫描.
  - b. 有序数组
    优点:单个查找和区间查找非常快
    缺点:插入数组需要移动后面的数组,效率非常差.
  - c. bTree:数据结构是N叉索引树
  > 以InnoDB的一个整数字段索引为例，这个N差不多是1200。这棵树高是4的时候，就可以存1200的3次方个值，这已经17亿了。考虑到树根的数据块总是在内存中的，一个10亿行的表上一个整数字段的索引，查找一个值最多只需要访问3次磁盘。其实，树的第二层也有很大概率在内存中，那么访问磁盘的平均次数就更少了。

##### InnoDB就是用BTree.
  至少有一棵树:以主键索引(聚簇索引)建立的.没有主键索引会默认一个主键索引
  也有非主键索引(二级索引)建立的.
  主键索引叶子结点包含所有字段记录.二级索引叶子结点只有主键索引数据,需要回表查询.
  
##### 名词解释
  - 页分裂:插入数据,当一个结点的数据页已经满了,就会申请新的数据页,然后挪动部分数据过去
  - 页合并:删除数据,当数据页数据量过少,就会叶合并.
  - 回表:二级索引查到会主键索引查询数据
  - 覆盖索引,select id from T where index = 1; 根据二级索引查询,但只需要主键索引数据,不需要回表
  - 最左前缀原则：B+Tree这种索引结构，可以利用索引的"最左前缀"来定位记录
只要满足最左前缀，就可以利用索引来加速检索。
最左前缀可以是联合索引的最左N个字段，也可以是字符串索引的最左M个字符
第一原则是：如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的。
  - 索引下推：在MySQL5.6之前，只能从根据最左前缀查询到ID开始一个个回表。到主键索引上找出数据行，再对比字段值。
MySQL5.6引入的索引下推优化，可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。


#### 普通索引和唯一索引选择
系统表空间就是用来放系统信息的，比如数据字典什么的，对应的磁盘文件是ibdata1.  
数据表空间就是一个个的表数据文件，对应的磁盘文件就是 表名.ibd
 
选择普通索引还是唯一索引？  
对于查询过程来说：  
- a. 普通索引，查到满足条件的第一个记录后，继续查找下一个记录，知道第一个不满足条件的记录
- b. 唯一索引，由于索引唯一性，查到第一个满足条件的记录后，停止检索   

但是，两者的性能差距微乎其微。因为InnoDB根据数据页来读写的。  

概念：change buffer  
当需要插入(非唯一索引,一定插入成功)一个数据页，如果数据页在内存中就直接插入，如果不在内存中，在不影响数据一致性的前提下，InnoDB会将这些更新操作缓存在change buffer中。下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行change buffer中的与这个页有关的操作。

change buffer是可以持久化的数据。在内存中有拷贝，也会被写入到磁盘上

purge:将change buffer中的操作应用到原数据页上，得到最新结果的过程，成为purge
访问这个数据页会触发purge，系统有后台线程定期purge，在数据库正常关闭的过程中，也会执行purge

唯一索引的更新不能使用change buffer

change buffer用的是buffer pool里的内存，change buffer的大小，可以通过参数innodb_change_buffer_max_size来动态设置。这个参数设置为50的时候，表示change buffer的大小最多只能占用buffer pool的50%。

将数据从磁盘读入内存涉及随机IO的访问，是数据库里面成本最高的操作之一。
change buffer 因为减少了随机磁盘访问，所以对更新性能的提升很明显。

change buffer使用场景
在一个数据页做purge之前，change buffer记录的变更越多，收益就越大。
对于写多读少的业务来说，页面在写完以后马上被访问到的概率比较小，此时change buffer的使用效果最好。这种业务模型常见的就是账单类、日志类的系统。

反过来，假设一个业务的更新模式是写入之后马上会做查询，那么即使满足了条件，将更新先记录在change buffer,但之后由于马上要访问这个数据页，会立即触发purge过程。
这样随机访问IO的次数不会减少，反而增加了change buffer的维护代价。所以，对于这种业务模式来说，change buffer反而起到了副作用。

索引的选择和实践：
尽可能使用普通索引。
redo log主要节省的是随机写磁盘的IO消耗(转成顺序写)，而change buffer主要节省的则是随机读磁盘的IO消耗。

#### 选错索引
索引的基数:explain得出的影响条数,使用采样统计
采样统计的时候，InnoDB默认会选择N个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数。

而数据表是会持续更新的，索引统计信息也不会固定不变。所以，当变更的数据行数超过1/M的时候，会自动触发重新做一次索引统计。

在MySQL中，有两种存储索引统计的方式，可以通过设置参数innodb_stats_persistent的值来选择：

设置为on的时候，表示统计信息会持久化存储。这时，默认的N是20，M是10。
设置为off的时候，表示统计信息只存储在内存中。这时，默认的N是8，M是16。


有时优化器会选错索引,当出现这种情况可以用
1.force index强行选择一个索引
2.考虑修改语句，引导MySQL使用我们期望的索引
3.在有些场景下，我们可以新建一个更合适的索引，来提供给优化器做选择，或删掉误用的索引。

#### 字符串加索引
1. 直接创建完整索引，这样可能比较占用空间；
2. 创建前缀索引，节省空间，但会增加查询扫描次数，并且不能使用覆盖索引；
3. 倒序存储，再创建前缀索引，用于绕过字符串本身前缀的区分度不够的问题；
4. 创建hash字段索引，查询性能稳定，有额外的存储和计算消耗，跟第三种方式一样，都不支持范围扫描。