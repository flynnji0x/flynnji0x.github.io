+++
title = "GeekTime -《MySQL实战45讲》笔记（三）"
date = '2022-04-15'
tags = ["笔记", "极客时间", "数据库", "MySQL"]
categories = ["极客时间"]
+++

### 25 Mysql是怎么保证高可用的？

- 主备延迟的最直接表现是，备库消费中转日志relay log的速度，比主库生产binlog的速度要慢
- 主备延迟的来源
    - 有些部署条件下，备库所在机器的性能要比主库所在的机器性能差
    - 备库压力大，备库的查询消耗大量的CPU资源，影响同步资源，造成主备延迟
    - 大事务，一次性操作的数据太多
    - 备库的并行复制能力
- 可靠性优先策略，主备延迟为0时才执行主从切换
- 可用性优先策略，不等主备数据同步完成，直接把连接切到备库
### 26 备库为什么会延迟好几个小时？

- 如果备库执行日志的速度持续低于主库生成日志的速度，将造成主备延迟过高的问题
- 在mysql5.6之前只支持单线程复制，为了解决主备延迟问题，进而演进为多线程复制
- 并行复制策略
    - 按表分发策略
    - 按行分发策略
        - 相比于按表并行分发策略，按行并行策略在决定分发的时候，需要消耗更多的Tech资源
    - mysql5.6版本的并行复制策略，支持的粒度是按库并行
    - mariaDB的并行复制策略，基于redo log的组提交特性
    - mysql5.7的并行复制策略
    - mysql5.7.22的并行复制策略
        - binlog-transcation-dependency-tracking参数，用于控制使用哪个策略
            - COMIT_ORDER
            - WRITESET
            - WRITESET_SESSION
### 27 主库出问题了，从库怎么办？

- 基于位点的主备切换
    - 主备切换时，位点很难精确定位到，故只能找一个稍微往前的，然后通过判断跳过那些在从库上已经执行过的事务
    - 在主备切换时，要主动跳过错误，有两种常用的方法：
        - 主动跳过事务
```sql
set global sql_salve_skip_counter = 1;
start slave;
```
        - 设置slave_skip_errors参数，直接设置跳过指定的错误，常见的有以下两类错误：
            - 1062，插入数据时的唯一键冲突
            - 1032，删除数据时找不到行，
- GTID，global transaction identifier，全局事务ID，mysql5.6版本引入的，是一个事务在提交的时候生成的，是这个事务的唯一标识
```sql
GTID=server_uuid:gno
```
    - 基于GTID的主备切换
```sql
master_auto_position=1
```
### 28 读写分离有哪些坑？

- 由于主从可能存在延迟，客户端执行完一个更新事务后马上发起查询，如果查询的是从库，就可能读到刚刚的事务更新之前的状态，称为过期读
- 处理过期读的方案：
    - 强制走主库方案
    - sleep方案
    - 判断主备无延迟方案
    - 配合semo-sync方案
    - 等主库位点方案
    - 等GTID方案
### 29 如何判断一个数据库是不是出问题了？

- 判断主库是否出问题？
    - select 1
        - 成功返回，只能说明这个库的进程还在，并不能说明主库没问题
        - 并发连接，show processlist的结果中的连接指的是并发连接
        - 并发查询，当前正在执行的语句，是指的并发查询（并发线程）
        - 查询线程加入锁等待之后，并发线程的计数会减一，也就是说等行锁的线程不算在并发线程限制数中
    - 查表判断
        - 一般做法是，在系统库中创建一个表，如health_check，然后定期执行select * from mysql.health_check
        - 如果数据库空间满了，这个方法就不好使
    - 更新判断
        - 在表中存入多行数据，并用主库，备库的server_id做主键
```sql
CREATE TABLE `health_check` (
 `id` int(11) NOT NULL,
 `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
 PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```
        - 然后更新语句，update mysql.health_check set t_modified=now()；
        - 相对比较常用，但是存在服务器IO资源分配导致的update检测判断错误的问题
    - 内部统计
        - 读取performance_schema库的file_summary_by_event_name表，统计了每次IO请求的时间
        - 打开所有performance_schema项，系统性能大概会下降10%，建议只打开自己需要的项进行统计
        - 通过判断MAX_TIMER的值来判断数据库是否出问题，通过设定阈值，检测超过阈值则数据库当前异常
    - 优先考虑update系统表方法，再配合增加检测performance_schema的统计项
### 30 用动态的观点看加锁
### 31 误删数据后除了跑路，还能怎么办？

- 删除数据分类以及处理方法
    - delete语句误删数据行
        - 可以用Flashback工具通过闪回把数据恢复回来，原理是修改binlog的内容，拿到原库重放
        - 不建议直接在主库上执行恢复操作，通过临时库恢复确认后再恢复回主库
        - 重要的是做到事前预防，如何事前预防的两个建议
            - sql_safe_updates参数设置为on，执行delete和update语句没where条件则会报错
            - 代码上线前，必须经过SQL审计
    - drop table或者truncate table语句误删数据表
        - 误删表恢复思路一般是，通过全量备份，再加应用binlog的方式
    - drop database语句误删数据库
        - 恢复思路同误删表
    - rm命令误删整个mysql实例
        - 对于mysql集群来说，删除一个实例，则在被删除的节点上恢复数据，然后重新接入集群
### 32 为什么还有kill不掉的语句？

- 两个kill命令
    - kill query + 线程id
        - 表示终止这个线程中正在执行的语句
    - kill connection + 线程id，connection可缺省
        - 表示断开这个线程的连接，如果语句正在执行，要先停止正在执行的语句
- 当执行kill query thread_id_B时，mysql处理kill命令的线程做了两件事
    - 把session b的运行状态改成THD::KILL_QUERY（将变量killed赋值为THD::KILL_QUERY）；
    - 给session b的执行线程发一个信号
- kill命令不生效的情况
    - 线程没有执行到判断线程状态的逻辑
    - 终止逻辑耗时较长的情况，需要等到终止逻辑完成，语句才算真正完成
        - 超大事务执行期间被kill
        - 大查询回滚时被kill
        - DDL命令执行到最后阶段被kill
- 关于客户端的误解
    - 如果库里面的表特别多，连接就会很慢
        - 原因其实是，我们感知到的连接过程慢，其实并不是连接慢，也不是服务端慢，而是客户端慢
        - 使用默认参数连接的时候，mysql客户端会提供一个本地库名和表名补全的功能，为此客户端多做了一些操作：
            - 执行show databases；
            - 切到对应库，执行show tables；
            - 将两个命令的结果用于构建一个本地的哈希表
    - -quick参数
        - 作用
            - 跳过表名自动补全功能
            - 调用的是mysql_use_result方法，mysql_store_result方法在查询结果太大的情况下会耗费较多的本地内存，可能会影响到客户端本地机器的性能
            - 不把执行命令记录到本地的命令历史文件
        - 所以，-quick参数的意思，是让客户端变得更快，而不是让服务端更快
### 33 我查这么多数据，会不会把数据库内存打爆？

- 概念：MySQL是边读边发的。
    - 这意味着，如果客户端接收很慢，会导致mysql服务端由于结果发不出去，事务的执行时间变长
- 对于正常的线上业务来说，如果一个查询的返回结果不会很多的话，我都建议你使用mysql_store_result这个接口，直接把查询结果保存到本地内存
- “Sending data”并不一定是指正在发送数据，也可能是处于执行器过程中的任意阶段
- InnoDB的Buffer Pool内存管理用的是LRU算法，算法核心是淘汰最久未使用的数据，InnoDB是用链表来实现的
- 对全表扫描操作量身定制，InnoDB对LRU算法策略进行了改进，将LRU链表分成了young区域和old区域
- 改进的LRU算法，最大的收益，是在全表扫描大表过程中，也用到了Buffer Pool，但对young区域完全没影响，从而保证了Buffer Pool响应正常业务的查询命中率
### 34 到底可不可以使用join？

- Index Nested-Loop Join，简称NLJ
![Index Nested-Loop Join 示意图](/image/GeekTime-《MySQL实战45讲》笔记（三）/2021-06-16_215255.png)

- 怎么选择驱动表
    - 使用join语句的话，需要让小表做驱动表，前提是被驱动表用得上索引
- Simple Nested-Loop Join
- Block Nested-Loop Join
![Block Nested-Loop Join 示意图](/image/GeekTime-《MySQL实战45讲》笔记（三）/2021-06-16_215611.png)

- join_buffer的大小是由参数join_buffer_size设定的，默认值是256k，如果放不下表的所有数据，策略是分段放
- 能不能使用join语句
    - 如果可以使用Index Nested-Loop Join算法，使用join语句是没问题的
    - 如果使用Block Nested-Loop Join算法，扫描行数就会过多，需要扫描被驱动表很多次，占用大量的系统资源，这种join尽量不要用
- 选择大表还是小表作驱动表
    - 在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与join的各个字段的总数据量，数据量小的那个表，就是小表，应该作为驱动表

