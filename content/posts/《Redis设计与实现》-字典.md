+++
title = "《Redis设计与实现》- 字典"
date = '2020-04-13'
tags = ["redis", "缓存中间件"]
categories = ["redis"]
id = "3729a742-fb41-1777-babc-72ae80ab24ec"
+++
### 字典

- 哈希表的定义

  ```c
  typedef struct dictht {
  		// 哈希表数组
      dictEntry **table;
  		// 哈希表大小
      unsigned long size;
  		// 哈希表大小掩码，用户计算索引值
  		// 总是等于size-1
      unsigned long sizemask;
  		// 该哈希表已有节点的数量
      unsigned long used;
  } dictht;
  ```

- table成员是一个数组，数组中的每个元素都是一个指向dictEntry结构的指针

- 哈希表节点定义

  ```c
  typedef struct dictEntry {
  		// 键
      void *key;
  		// 值
      union {
          void *val;
          uint64_t u64;
          int64_t s64;
          double d;
      } v;
  		// 指向下个哈希表节点，形成链表
      struct dictEntry *next;
  } dictEntry;
  ```

- next属性是指向另一个哈希表节点的指针，用来解决哈希冲突的问题

- 字典的定义

  ```c
  typedef struct dict {
  		// 类型特定函数
      dictType *type;
  		// 私有数据
      void *privdata;
  		//哈希表
      dictht ht[2];
  		// rehash索引
      long rehashidx; /* rehashing not in progress if rehashidx == -1 */
      int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
  } dict;
  ```

- ht中每一项都是一个dictht哈希表，一般情况下，字典只使用ht[0]哈希表，ht[1]只会在对ht[0]进行rehash时使用

- type属性是一个指向dictType结构的指针

  ```c
  typedef struct dictType {
  		// 计算哈希值的函数
      uint64_t (*hashFunction)(const void *key);
  		// 复制键的函数
      void *(*keyDup)(void *privdata, const void *key);
  		// 复制值的函数
      void *(*valDup)(void *privdata, const void *obj);
  		// 对比键的函数
      int (*keyCompare)(void *privdata, const void *key1, const void *key2);
      // 销毁键的函数
  		void (*keyDestructor)(void *privdata, void *key);
      // 销毁值的函数
  		void (*valDestructor)(void *privdata, void *obj);
      int (*expandAllowed)(size_t moreMem, double usedRatio);
  } dictType;
  ```

- 当添加一个新的键值对到字典里面时，通过键-》哈希值-》索引值，根据计算得到索引值将键值对放到哈希表数组的指定索引上面

- 当字典被用作数据库的底层实现，或哈希键的底层实现，Redis使用MurmurHash2算法来计算键的哈希值

- Redis使用链地址法来解决哈希冲突

- 哈希表键值对的增多或减少，哈希表的负载因子load factor会发生变化，通过对哈希表的大小进行相应的扩展或收缩，将load factor维持在一个合理范围内

- rehash

  - 渐进式rehash
    - 将ht[0]的每个索引值的rehash工作分摊到每次对字典执行添加，输出，查找或更新操作中
    - 在渐进式rehash过程中，新的键值对只会保存到ht[1]中，而ht[0]不在进行如何添加操作，保证了ht[0]的键值对数量只减不增，并最终ht[0]变成空表