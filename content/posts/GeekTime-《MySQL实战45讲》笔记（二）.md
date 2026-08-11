+++
title = "GeekTime -《MySQL实战45讲》笔记（二）"
date = '2022-04-03'
tags = ["笔记", "极客时间", "数据库", "MySQL"]
categories = ["极客时间"]
+++
# MySQL实战45讲（二）

### 11 怎么给字符串加索引

- 使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本
- 建立索引时关注的是区分度，区分度越高越好
- 使用覆盖所有就用不上覆盖索引对查询性能的优化了，这是在选择是否使用前缀索引时需要考虑的一个因素
- 前缀索引区分度不够好的其他方式
    - 倒序存储
    - 使用hash字段
- 字符串创建索引的方式：
    - 完整索引
    - 前缀索引
    - 倒序存储
    - hash字段索引
### 12 为什么我的Mysql会”抖“一下？

- 当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内容页为”脏页“，内存数据写入到磁盘后，内存和磁盘上的数据页的内容就一致了，称为”干净页“
- Mysql偶尔抖一下的那个瞬间，可能就是在刷脏页
- 什么情况会引发数据库的flush过程
    - redo log写满了
    - 系统内存不够，淘汰一些数据页
    - Mysql认为系统空闲时
    - Mysql正常关闭服务时
- InnoDB用缓存池 buffer pool 管理内存
- InnoDB刷脏页的控制策略
    - innodb_io_capacity，告诉InnoDB你的磁盘能力
- 平时多关注脏页比例，不要让它经常接近75%
- innodb_flush_neighbors
### 13 为什么表数据删掉一半，表文件大小不变？

- 参数innodb_file_per_table
- innodb删除行记录，只是把记录标记为删除，之后会复用这个位置，但磁盘大小不会缩小
- innodb删除数据页上的所有记录，整个数据页就可以被复用了
- 用delete命令，删除整个表的数据，所有数据页都会被标记为可复用，但是磁盘上文件大小不会变小
- 重建表
    - 使用alter table A engine=InnoDB来重建表
- Mysql5.6版本开始引入的Online DDL，对重建表流程做了优化
- Online 和 inplace
    - DDL过程如果是Online的，就一定是inplace的
    - 反过来未必，也就是说inplace的DLL，有可能不是DDL的，如，加全文索引FULLTEXT index和空间索引 SPATIAL index
- 三种方式重建表
    - alter table t engine = InnoDB，也即recreate
    - analyze table t
    - optimize table t 等于recreate+analyze
### 14 count（*）这么慢，我该怎么办？

- count（*）的实现方式
    - MyISAM引擎把一个表的总行数存在了磁盘上，执行count（*）的时候会直接返回这个数，效率很高；
    - InnoDB执行时，需要把数据一行一行从引擎里面读出来，然后累计计数
- 在保证正确的前提下，尽量减少扫描的数据量，是数据库系统设计的通用法则之一
- 不同的count用法
    - count（主键id）
    - count（1）
    - count（字段）
    - count（*），不会把全部字段取出来，专门做了优化，不取值
- 按效率排序
    - count（字段）<count（主键id）<count（1）约等于count（*）
### 16 order by是怎么工作的？
    - 全字段排序
‘using filesort’表示需要排序，Mysql会给每个线程分配一块内存用于排序，称为sort_buffer
        - sort_buffer_size，表示的是Mysql为排序开辟的内存大小，如果排序数据大<sort_buffer_size，则在内存中完成，反之则利用磁盘临时文件辅助排序
        - number_of_tmp_files，表示的是，排序过程中使用的临时文件数
    - rowid排序
        - max_length_for_sort_data，是Mysql专门控制用于排序的行数据的长度的一个参数，如果单行出超过这个值，则Mysql认为要换一个排序算法
    - Mysql认为内存太小，才会采用rowid排序算法，如果认为内存足够大，会优先选择全字段排序，直接把需要的字段放到sort_buffer中
    - 覆盖索引，索引上的信息足够满足查询请求，不需要回到主键索引上取数据
    - 索引是有维护代价的，是不是需要用上覆盖索引，是需要权衡的决定
### 17 如何正确地显示随机消息
    - 使用order by rand()实现随机获取，使用了内存临时表，内存临时表排序的时候使用了rowid排序方法，执行代价比较大，在设计的时候要尽量避开这中写法
```sql
select word from words order by rand() limit 3;
```
        - Extra字段中，Using temporary，表示需要使用临时表；Using filesort，表示需要执行排序操作
- 对于InnoDB表来说，相对于rowid排序，执行全字段排序会减少磁盘访问，因此会被优先选择
- Mysql的表是用什么方法定位一行数据的？表的主键，或者是InnoDB会自己生成一个长度为6字节的rowid作为主键
- tmp_table_size配置限制了内存临时表的大小，如果临时表超过了，则内存临时表会转成磁盘临时表
- 磁盘临时表引擎默认是InnoDB，由参数internal_disk_storage_engine控制
- Mysql5.6引入了一个新的排序算法，即优先队列排序算法
- 如何正确的随机排序？
### 18 为什么这些SQL语句逻辑相同，性能却差异巨大？

- 对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能
- 案例：
    - 条件字段函数操作
    - 隐式类型转换，如字符串和数字做比较时，字符串转换为整数
    - 隐式字符编码转换
### 19 为什么我只插一行的语句，也执行这么慢？

- 常见的情况：
    - 查询长时间不返回。一般大概率是查询表被锁住了
        - 等MDL锁，waiting for table metadata lock，表示有一个线程正在表上请求或持有MDL写锁，把select语句堵住了。查询sys.schema_table_lock_waits可以找到造成阻塞的process id
        - 等flush
        - 等行锁
    - 查询慢
### 20 幻读是什么，幻读有什么问题？

- 幻读，指的是一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行
- 幻读的问题
    - 语义问题
    - 数据一致性问题
- 即使那所有的记录都加上锁，还是阻止不了新插入的记录
- InnoDB通过引入新的锁，间隙锁，gap lock，来解决幻读问题
- 间隙锁，锁的是两个值之间的空隙
- 行级锁，是不同类型行级锁之间有冲突，而间隙锁则是，和“往这个间隙插入一个记录”这个操作存在冲突关系，间隙锁之间不存在冲突关系
- 间隙锁和行级锁合称next-key lock，每个lock都是前开后闭区间，而间隙锁是开区间
- 间隙锁只在可重复读隔离级别下才有效
### 21 为什么我只改一行的语句，锁这么多？
### 22 Mysql有哪些饮鸩止渴提高性能的方法？

- 短连接风暴导致的性能问题，max_connections参数，用来控制一个Mysql实例同时存在的连接数的上限，超过这个值，系统会拒绝接下来的连接请求，提示，too many connections
- 可能的解决方案
    - 先处理掉那些占着连接但是不工作的线程
    - 减少连接过程的消耗
- 慢查询性能问题
    - 索引没有设计好
        - 变更索引，使用alter table
    - SQL语句没写好
        - 使用query_write功能，把输入的语句改写成另外一种模式
    - Mysql选错了索引
        - 应急方案是给语句加force index
- QPS突增问题
    - 修改白名单
    - 将用户删除，断开当前的连接
    - 可以使用query_write功能，先将压力最大的语句直接改写为select 1返回
### 23 Mysql是怎么保证数据不丢的？

- binlog的写入机制
    - 写入流程：binlog cache -> binlog files -> disk
    - 每个线程有自己的binlog cache，write到binlog files指的是写入到文件系统的page cache，并没有持久化
    - fsync到disk才是将数据持久化到磁盘的操作
- write和fsync的时机，是由参数sync_binlog控制的
- 在IO瓶颈的场景里，将sync_binlog设置成一个比较大的值，可以提升性能，但对应的风险：如果主机发生异常重启，会丢失最近N个事务的binlog日志
- redo log的写入机制
    - redo log的三种状态
        - 存在redo log buffer中，即Mysql进程的内存中
        - 已write，未fsync，存在文件系统的page cacge中
        - 已fsync，存在磁盘中
    - 未提交事务的redo log在以下三种场景中有可能会被写入到磁盘中：
        - 后台线程每秒一次的轮询操作
        - redo log buffer占用的空间即将达到innodb_log_buffer_size一半时，后台线程会主动写盘
        - 并行的其他事务提交时，顺带将这个事务的redo log buffer持久化到磁盘
### 24 Mysql是怎么保证主备一致的？

- readonly对超级权限用户是无效的，而主备同步更新的线程就拥有超级权限
- binlog的三种格式
    - statement
原封不动地记录执行的sql语句，但是因为索引原因，可能导致主从执行结果不一致
    - row
记录了执行的行记录，占用大量空间，耗费IO资源，影响执行速度
    - mixed（前两种格式的混合）
Mysql会判断执行SQL使用哪种格式保存binlog，既利用了statement格式的优点，也避免了数据不一致的风险
- 如果线上Mysql设置的binlog格式是statement，基本上就是不合理的设置，至少要设置为mixed
- 用row的好处：恢复数据，delete语句可以改为insert语句，insert语句改为delete语句，而update语句可以将event前后的信息对调一下，重新执行即可
- 双master模式下，如何解决两个节点间的循环复制问题？
    - 1，规定两个库的server_id不同，如果相同，则不能互为主备关系
    - 2，备库拿到主库的binlog执行并生成与原binlog的server_id相同的新的binlog
    - 3，每个库收到日志后，先判断server_id，如果和自己的相同，表示是自己生成的，直接丢弃这个日志
