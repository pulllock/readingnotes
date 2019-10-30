# 顺序消息

rocketmq有两种顺序级别：

- 普通顺序消息，producer将相关联的消息发送到相同的消息队列。
- 完全严格顺序，在普通顺序消费的基础上，consumer严格顺序消费。

普通顺序消费，需要我们自己提供MessageQueueSelector来选择具体的消息队列，要自己保证能选择到同一个队列的算法。

严格顺序消费，需要有三把锁来保证严格顺序消费：

- Broker消息队列锁，分布式锁。在集群模式下Consumer从Broker获得该锁后，才能进行消息拉取、消费。广播模式下，Consumer无需该锁。
- Consumer消息队列锁，本地锁。Consumer获得该锁才能操作消息队列。
- Consumer消息处理队列消费锁，本地锁。Consumer获得该锁才能消费消息队列。