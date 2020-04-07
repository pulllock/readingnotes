MySQL提供插件式的表存储引擎，常用的存储引擎有：InnoDB、MyISAM。

# InnoDB和MyISAM简介

## InnoDB存储引擎

- 支持事务
- 支持行锁
- 支持外键
- 支持非锁定读，默认读取操作不加锁
- 使用MVCC（多版本并发控制）实现并发
- 实现SQL的4中隔离级别，默认REPEATABLE级别
- 使用next-key locking策略避免幻读
- 数据存储使用聚集索引，表的数据都是按照主键的顺序进行存放
- 没有指定主键时，InnoDB会默认为每一行生成一个6字节的ROWID，作为主键
- 不保存表的总行数，count(*)会全表扫描来统计

## MyISAM存储引擎

- 不支持事务
- 只支持表锁
- 支持全文引擎
- 保存表的总行数，count(*)直接返回总数

# InnoDB存储引擎

## InnoDB后台线程

后台线程主要负责刷新内存中的数据，保证内存缓存的是最新数据，还需要将已修改数据同步到磁盘文件，保证数据库发生异常时InnoDB能恢复到正常运行状态。

- Master Thread，将缓冲区中的数据异步刷新到磁盘，包括脏页刷新、合并插入缓冲、UNDO页回收。
- IO Thread，InnoDB使用AIO处理写IO请求
- Purge Thread，事务提交后，undolog可能不再需要，该线程可以回收已分配的undo页
- Page Cleaner Thread，脏页刷新操作

## 内存

内存中的页同步到磁盘的操作，不是每次更新都触发，而是基于Checkpoint的机制刷新回磁盘。

内存中数据页类型有：

- 索引页
- 数据页
- undo页
- 插入缓冲
- 自适应哈希索引
- 锁信息
- 数据字典信息

# 内存管理

- LRU List
- Free List
- Flush List

使用LRU算法管理内存，InnoDB在LRU列表中加入了midpoint位置，读取到的新页不是直接放到LRU首部，而是放到LRU的midpoint位置。

LRU列表中的页被修改后，成为脏页，会通过Checkpoint机制将脏页刷新回磁盘，Flush列表中的页就是脏页列表，脏页在LRU列表和Flush列表中都存在。

## redo log buffer

重做日志缓冲区，InnoDB会先将重做日志放到这个缓冲区，在按照一定频率刷新到重做日志文件。

- Master Thread每一秒将重做日志刷新到重做日志文件中
- 每个事务提交会将重做日志缓存刷新到重做日志文件中
- 重做日志缓冲区剩余空间小于二分之一时将重做日志缓存刷新到重做日志文件中

## Checkpoint技术

InnoDB不是在每次操作了页记录就刷新到磁盘，而是使用Checkpoint技术来刷新缓存。