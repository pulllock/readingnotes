1. 数据类型支持不同
	
	Memcached仅支持简单的key-value结构.
	Redis支持数据类型更丰富,String,Hash,List,Set,SortedSet

2. 内存管理机制不同
	
	Redis并不是所有的数据都一直存储在内存中.这是和Memcached相比一个最大的区别。当物理内存用完时，Redis可以将一些很久没用到的value交换到磁盘。Redis只会缓存所有的key的信息，如果Redis发现内存的使用量超过了某一个阀值，将触发swap的操作，Redis根据“swappability = age*log(size_in_memory)”计算出哪些key对应的value需要swap到磁盘。然后再将这些key对应的value持久化到磁盘中，同时在内存中清除。这种特性使得Redis可以保持超过其机器本身内存大小的数据。
	
	在默认的情况下，Redis会出现阻塞
	
	Memcached默认使用Slab Allocation机制管理内存，其主要思想是按照预先规定的大小，将分配的内存分割成特定长度的块以存储相应长度的key-value数据记录，以完全解决内存碎片问题。Slab Allocation机制只为存储外部数据而设计，也就是说所有的key-value数据都存储在Slab Allocation系统里，而Memcached的其它内存请求则通过普通的malloc/free来申请，因为这些请求的数量和频率决定了它们不会对整个系统的性能造成影响
	
	Memcached的内存管理制效率高，而且不会造成内存碎片，但是它最大的缺点就是会导致空间浪费。因为每个Chunk都分配了特定长度的内存空间，所以变长数据无法充分利用这些空间
	
	Redis为了方便内存的管理，在分配一块内存之后，会将这块内存的大小存入内存块的头部
	
	总的来看，Redis采用的是包装的mallc/free，相较于Memcached的内存管理方法来说，要简单很多

3. 数据持久化支持

	Redis虽然是基于内存的存储系统，但是它本身是支持内存数据的持久化的，而且提供两种主要的持久化策略：RDB快照和AOF日志。而memcached是不支持数据持久化操作的。

	对于一般性的业务需求，建议使用RDB的方式进行持久化，原因是RDB的开销并相比AOF日志要低很多，对于那些无法忍数据丢失的应用，建议使用AOF日志。

4. 集群管理的不同
	
	Memcached本身并不支持分布式，因此只能在客户端通过像一致性哈希这样的分布式算法来实现Memcached的分布式存储
	
	最新版本的Redis已经支持了分布式存储功能。Redis Cluster是一个实现了分布式且允许单点故障的Redis高级版本，它没有中心节点，具有线性可伸缩的功能

# 参考

[http://www.biaodianfu.com/redis-vs-memcached.html](http://www.biaodianfu.com/redis-vs-memcached.html)