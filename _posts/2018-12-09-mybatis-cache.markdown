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

可以看出每个Executor的local cache都是存在同一个sqlSession里面。不同sqlSession下面的cache不能共享。
    
    public abstract class BaseExecutor implements Executor {
    
      private static final Log log = LogFactory.getLog(BaseExecutor.class);
    
      protected Transaction transaction;
      protected Executor wrapper;
    
      //延迟加载队列（线程安全）
      protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
      //本地缓存机制（Local Cache）防止循环引用（circular references）和加速重复嵌套查询(一级缓存)
      //本地缓存
      protected PerpetualCache localCache;
      //本地输出参数缓存
      protected PerpetualCache localOutputParameterCache;
      protected Configuration configuration;
在Executor里面保存着一个localCache变量。这个变量集成几口Cache，提供基本的put和get操作。背后的存储是用hashMap。
而缓存的key是一个cacheKey的类，看看他的update方法和equals方法
    
    public class CacheKey implements Cloneable, Serializable {
    
      private static final long serialVersionUID = 1146682552656046210L;
    
      public static final CacheKey NULL_CACHE_KEY = new NullCacheKey();
    
      private static final int DEFAULT_MULTIPLYER = 37;
      private static final int DEFAULT_HASHCODE = 17;
    
      private int multiplier;
      private int hashcode;
      private long checksum;
      private int count;
      private List<Object> updateList;
    
      public CacheKey() {
        this.hashcode = DEFAULT_HASHCODE;
        this.multiplier = DEFAULT_MULTIPLYER;
        this.count = 0;
        this.updateList = new ArrayList<Object>();
      }
      private void doUpdate(Object object) {
        //计算hash值，校验码
        int baseHashCode = object == null ? 1 : object.hashCode();
    
        count++;
        checksum += baseHashCode;
        baseHashCode *= count;
    
        hashcode = multiplier * hashcode + baseHashCode;
    
        //同时将对象加入列表，这样万一两个CacheKey的hash码碰巧一样，再根据对象严格equals来区分
        updateList.add(object);
      }
      @Override
      public boolean equals(Object object) {
        if (this == object) {
          return true;
        }
        if (!(object instanceof CacheKey)) {
          return false;
        }
    
        final CacheKey cacheKey = (CacheKey) object;
    
        //先比hashcode，checksum，count，理论上可以快速比出来
        if (hashcode != cacheKey.hashcode) {
          return false;
        }
        if (checksum != cacheKey.checksum) {
          return false;
        }
        if (count != cacheKey.count) {
          return false;
        }
    
        //万一两个CacheKey的hash码碰巧一样，再根据对象严格equals来区分
        //这里两个list的size没比是否相等，其实前面count相等就已经保证了
        for (int i = 0; i < updateList.size(); i++) {
          Object thisObject = updateList.get(i);
          Object thatObject = cacheKey.updateList.get(i);
          if (thisObject == null) {
            if (thatObject != null) {
              return false;
            }
          } else {
            if (!thisObject.equals(thatObject)) {
              return false;
            }
          }
        }
        return true;
      }
可以看出通过吧sql语句通过hashcode做了一序列的运算得到一个key。保证两个不同的sql是不同的key。而保证缓存的唯一性。
总结：mybatis的一级缓存只是简单的hashmap结构，并且只能在同一个sqlSession生效，他是尽量保证相同一个业务里面避免相同的sql数据库穿透。
##二级缓存

###设计模式

##引发对工程代码的思考


