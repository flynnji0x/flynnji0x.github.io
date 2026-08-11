+++
title = "《Redis设计与实现》- 跳跃表"
date = '2020-04-13'
tags = ["redis", "缓存中间件"]
categories = ["redis"]
id = "3729a742-fb41-4777-babc-73ae80ab24ec"
+++
### 跳跃表

- 跳跃表，skiplist，是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的

- 在大部分情况下，跳跃表的效率可以和平衡树相媲美，而且实现比平衡树更为简单

- 跳跃表节点定义

  ```c
  typedef struct zskiplistNode {
      // 成员对象
  		sds ele;
  		// 分值
      double score;
  		// 后退指针
      struct zskiplistNode *backward;
  		// 层
      struct zskiplistLevel {
  	      // 前进指针
  				struct zskiplistNode *forward;
  				// 跨度
          unsigned long span;
      } level[];
  } zskiplistNode;
  ```

- 属性定义

  - 层
    - 每一个层都包含一个指向其他节点的指针，程序可以通过这些层来加快访问其他节点的速度，一般来说，层的数量越多，访问其他节点的速度就越快
  - 前进指针
    - level[i].forward，用于从表头向表尾方向访问节点
  - 跨度
    - level[i].span，用于记录两个节点之间的距离，用来计算排位，rank
  - 后退指针
    - 每个节点只有一个后退指针，每次只能后退至前一个节点
  - 分值和成员
    - 一个double类型的浮点数，所有节点都按分值从小到大来排序
    - 成员对象是一个指针，指向一个字符串对象，对象值是一个SDS值
    - 分值相同的节点，按成员对象的大小来正向排序

- 跳跃表的定义

  ```c
  typedef struct zskiplist {
  		// 表头节点和表尾节点
      struct zskiplistNode *header, *tail;
      // 表中节点的数量
  		unsigned long length;
  		// 表中层数最大的节点的层数
      int level;
  } zskiplist;
  ```

  - 属性定义
    - header和tail
      - 分别指向跳跃表的表头和表尾节点，程序定位表头节点和表尾节点的复杂度为O（1）
    - length
      - 可以在O（1）复杂度内返回跳跃表的长度
    - level
      - 用于在O（1）复杂度内获取跳跃表中层高最大的那个节点的层数量