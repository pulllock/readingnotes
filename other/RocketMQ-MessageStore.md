# 高性能存储
- 支持顺序写的消息存储结构 
- MappedFile 创建、异步线程结合 CountDownLatch 实现任务执行的异步转同步
- 堆外内存
- MappedFile 内存预热和JNA内存锁定
# MappedFile

TransientStorePool与MappedFile在数据处理上的差异在什么地方呢？分析其代码，TransientStorePool会通过ByteBuffer.allocateDirect调用直接申请对外内存，消息数据在写入内存的时候是写入预申请的内存中。在异步刷盘的时候，再由刷盘线程将这些内存中的修改写入文件。

那么与直接使用MappedByteBuffer相比差别在什么地方呢？修改MappedByteBuffer实际会将数据写入文件对应的Page Cache中，而TransientStorePool方案下写入的则为纯粹的内存。因此在消息写入操作上会更快，因此能更少的占用CommitLog.putMessageLock锁，从而能够提升消息处理量。使用TransientStorePool方案的缺陷主要在于在异常崩溃的情况下回丢失更多的消息。

http://soliloquize.org/2018/08/25/RocketMQ-CommitLog%E5%88%B7%E7%9B%98%E6%9C%BA%E5%88%B6/

# TransientStorePool

开启TransientStorePool后，master写入变成 先写堆外内存 ，然后批量commit到 FileChannel写入，而主从同步判断能同步到的消息是已经commit到FileChannel的消息，而消息由堆外内存commit到PageCache是有一定频率的，是受commitIntervalCommitLog，commitCommitLogThoroughInterval两个参数影响，默认值是200ms，所以你能看到消息大体都落在100ms-200ms之间。

# RocketMQ 消息存储
RocketMQ主要存储的文件包括CommitLog文件、ConsumeQueue文件、IndexFile文件。

- CommitLog：消息存储文件，所有主题的消息都存储在CommitLog文件中。
- ConsumeQueue：消息消费队列，消息到达CommitLog后，将异步转发到消息消费队列，供消息消费者消费。
- IndexFile：消息索引文件，主要存储消息Key与Offset对应关系。
- 事务状态服务：存储每条消息的事务状态。
- 定时消息服务：每一个延迟级别对应一个消息消费队列，存储延迟队列的消息拉取进度。

## 一些类说明
CommitLog : MappedFileQueue : MappedFile = 1 : 1 : N

- MappedFile 是硬盘上一个个文件的映射
- MappedFileQueue是MappedFile所在的文件夹，将MappedFile封装成文件队列，对上层提供可无限使用的文件容量
- CommitLog 针对MappedFileQueue的封装使用

CommitLog目前存储在MappedFile有两种类型：

1. MESSAGE：消息
2. BLANK：文件不足以存储消息时的空白占位

CommitLog存储在MappedFile的结构：

```
-----------------------------------------------------
| MESSAGE[1] | MESSAGE[2] | ... | MESSAGE[N] | BLANK|
-----------------------------------------------------
```




## RocketMQ存储文件

存储路径为`${ROCKET_HOME/store}`，存储的目录和文件有：

- commitlog，消息存储目录
- config，运行期间一些配置信息
- consumequeue，消息消费队列存储目录
- index，消息索引文件存储目录
- abort，如果存在abort文件，说明Broker非正常关闭，该文件默认启动时创建，正常退出前删除。
- checkpoint，文件监测点，存储commitlog文件最后一次刷盘时间戳、consumequeue最后一次刷盘时间、index索引文件最后一次刷盘时间戳。

### config目录下存放的文件

- consumerFilter.json，主题消息过滤信息
- consumerOffset.json，集群消费模式消息消费进度
- delayOffset.json，延时消息队列拉取进度
- subscriptionGroup.json，消息消费组配置信息
- topic.json，topic配置属性

