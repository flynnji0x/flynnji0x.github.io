+++
title = "《Redis设计与实现》- 对象"
date = '2020-04-13'
tags = ["redis", "缓存中间件"]
categories = ["redis"]
id = "3729a742-fb41-4771-babc-73ae80ab24ec"
+++
### 对象

- 每当新创建一个键值对时，会至少创建两个对象，一个是键对象。一个是值对象

- 每个对象由一个redisObject结构表示

  ```c
  typedef struct redisObject {
  		// 类型
      unsigned type:4;
  		// 编码
      unsigned encoding:4;
      unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                              * LFU data (least significant 8 bits frequency
                              * and most significant 16 bits access time). */
      // 引用计数
  		int refcount;
  		// 指向底层实现数据结构的指针
      void *ptr;
  } robj;
  ```

- 属性定义

  - type，redis中创建的键总是一个字符串对象，而值则可能是以下五种类型对象之一
    - REDIS_STRING
    - REDIS_LIST
    - REDIS_HASH
    - REDIS_SET
    - REDIS_ZSET
  - encoding，记录了对象所使用的编码，也即是说这个对象使用什么数据结构作为对象的底层实现
    - REDIS_ENCONDING_INT
      - long类型的整数
      - int
    - REDIS_ENCONDING_EMBSTR
      - embstr编码的简单动态字符串
      - embstr
    - REDIS_ENCONDING_RAW
      - 简单动态字符串
      - raw
    - REDIS_ENCONDING_HT
      - 字典
      - hashtable
    - REDIS_ENCONDING_LINKEDLIST
      - 双端链表
      - linkedlist
    - REDIS_ENCONDING_ZIPLIST
      - 压缩列表
      - ziplist
    - REDIS_ENCONDING_INTSET
      - 整数集合
      - intset
    - REDIS_ENCONDING_SKIPLIST
      - 跳跃表和字典
      - skiplist
  - 每种类型对象都至少使用了两种不同的编码
    - REDIS_STRING
      - REDIS_ENCONDING_INT
      - REDIS_ENCONDING_EMBSTR
      - REDIS_ENCONDING_RAW
    - REDIS_LIST
      - REDIS_ENCONDING_ZIPLIST
      - REDIS_ENCONDING_LINKEDLIST
    - REDIS_HASH
      - REDIS_ENCONDING_ZIPLIST
      - REDIS_ENCONDING_HT
    - REDIS_SET
      - REDIS_ENCONDING_INTSET
      - REDIS_ENCONDING_HT
    - REDIS_ZSET
      - REDIS_ENCONDING_ZIPLIST
      - REDIS_ENCONDING_SKIPLIST

- 内存回收

  - redis构建了一个引用计数reference counting技术实现的内存回收机制
  - 通过跟踪对象的引用计数信息，在适当的时候自动释放对象并进行内存回收
  - 对象的生命周期
    - 创建对象
    - 操作对象
    - 释放对象

- 对象共享

  - redis为了节约内存，让多个键共享同一个值对象
  - 数据库种保存的相同值对象越多，对象共享机制旧能节约越多的内存
  - 受到CPU时间的限制，redis只对包含整数值的字符串对象进行共享

- 对象的空转时长

  - lru属性记录了对象最后一次被命令程序访问的时间
  - 空转时长较高的键会优先被服务器释放，从而回收内存