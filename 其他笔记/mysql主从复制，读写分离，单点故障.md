# 读写分离
一般是做一个主从结构，一个主库负责写，多台从库负责读。

实现的方式有两种：

- 应用层实现，使用SpringJDBC，mybatis等的多数据源来进行访问数据库。
- 代理实现，在应用层和数据库之间加上一个代理服务，应用层访问代理，代理根据请求类型自动访问不同的数据库。

## 应用层实现
在应用层就已经决定了读写的方向，所有写操作写到主库，读操作到读库。

Spring JDBC可以定义多个数据源，我们可以定义一个主数据源和一个从数据源，主数据源负责写操作，从数据源负责读操作。但是这种办法如果要是有多个读数据源的话，就很难办了。可以继承AbstractRoutingDataSource实现自己的算法。

应用层实现读写分离，不需要底层复杂的配置，性能好，但是对应用的侵入性大，不利于扩展。

## 代理实现
代理方式实现主从分离，所有请求都集中到代理层，代理层根据请求类型，选择不同数据库服务器。

### mysql-proxy
mysql-proxy是mysql提供的基于代理的负载均衡。

安装好mysql-proxy之后，使用命令`mysql-proxy --proxy-backend-addresses 192.168.1.101:3306 --proxy-backend-addresses 192.168.1.102:3306`

### haproxy
haproxy需要在应用层做读写分离，haproxy需要配置读写端口。

## 驱动实现
使用ReplicationDriver可以实现读写分离。主读写，从只读。

ReplicationDriver是靠Connection的readonly属性来决定是否访问从库。ReplicationDriver和Driver两个类不能同时使用

url连接需要改为`jdbc:mysql:replication://192.168.1.101:3306,192.168.1.102:3306/xxxx?roundRobinLoadBalance=true&characterEncoding=UTF-8`

还可以实现负载均衡：`jdbc:mysql:loadbalance`

采用主主架构的时候，可以使用`jdbc:mysql:loadbalance`实现负载均衡，应用层读取任意一个节点不会出问题，因为是双向同步复制机制，但是一主一从或者一主多从使用`jdbc:mysql:loadbalance`就会出问题。

# 单点故障
解决单点故障可以通过主主架构加上keepalived来实现：主主架构，实现数据同步，使用keepalived虚拟ip，从网络层实现单点故障时ip自动切换，使用keepalived配置，实现读写分离。

还可以使用主主+LVS+keepalived来实现。

# Mysql集群
分布式mysql解决方案一般有两种：客户端实现和中间代理解决方案。

## 客户端实现
对业务有侵入。
## 中间代理解决方案
使用代理，对业务无侵入。


# mysql集群，事务管理