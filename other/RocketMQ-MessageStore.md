# RocketMQ 消息存储
RocketMQ主要存储的文件包括CommitLog文件、ConsumeQueue文件、IndexFile文件。

- CommitLog：消息存储文件，所有主题的消息都存储在CommitLog文件中。
- ConsumeQueue：消息消费队列，消息到达CommitLog后，将异步转发到消息消费队列，供消息消费者消费。
- IndexFile：消息索引文件，主要存储消息Key与Offset对应关系。
- 事务状态服务：存储每条消息的事务状态。
- 定时消息服务：每一个延迟级别对应一个消息消费队列，存储延迟队列的消息拉取进度。

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

