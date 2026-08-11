+++
title = "《Redis设计与实现》- 双端链表"
date = '2020-04-13'
tags = ["redis", "缓存中间件"]
categories = ["redis"]
id = "3729a742-fb41-4777-babc-72ae80ab24ec"
+++
### 链表

- 链表每一个节点listNode定义

  ```c
  typedef struct listNode {
  		// 前置节点
  		struct listNode *prev;
      // 后置节点
  		struct listNode *next;
  		// 节点的值
      void *value;
  } listNode;
  ```

- 多个listNode可以通过prev和next指针组成双端链表

- list结构提供更多的属性和方法

  ```c
  typedef struct list {
  		// 表头节点
      listNode *head;
  		// 表尾节点
      listNode *tail;
      // 节点复制函数
  		void *(*dup)(void *ptr);
  		// 节点值释放函数
      void (*free)(void *ptr);
  		// 节点值对比函数
      int (*match)(void *ptr, void *key);
      // 链表所包含的节点数量
  		unsigned long len;
  } list;
  ```

- Redis的链表实现的特性总结：

  - 双端
    - 获取某个节点的前置节点和后置节点复杂度都是O（1）
  - 无环
    - 表头节点的prev指针和表尾节点的next指针都指向null，对链表的访问已null为终点
  - 带表头指针和表尾指针
    - 获取链表的表头节点和表尾节点的复杂度为O（1）
  - 带链表长度计数器
    - 获取链表中节点数量的复杂度为O（1）
  - 多态
    - 节点用void*指针来保存节点值，可以用于保存各种不同类型的值