Redis实现分布式锁大概有几种方案：使用set命令同时指定过期时间和NX、使用lua脚本和setnx加过期时间配合、基于Redlock算法的Redisson实现。分布式锁除了可以使用Redis实现外，还可以使用其他的实现，比如：mysql数据库方式、Tair实现、Zookeeper等等

# set命令加过期时间加nx

Redis的set命令有一系列的选项：

```shell
set key value [EX seconds][PX milliseconds][NX|XX]
```

- EX 设置过期时间，单位秒
- PX 设置过期时间，单位毫秒
- NX 仅当key不存在时设置值
- XX 仅当key存在时设置值

# lua脚本加setnx加过期时间

Redis中可以执行lua脚本，脚本执行是原子性的，可以使用lua脚本中执行多个命令：

```lua
if redis.call('setnx',KEYS[1],ARGV[1]) == 1 then redis.call('expire',KEYS[1],ARGV[2]) return 1 else return 0 end
```

# 锁的释放

分布式锁的value需要每个获取锁的线程都设置对应唯一的值，防止不同线程误删锁，设置唯一的值后就可以在删除的时候验证下锁是不是属于当前线程。不能使用del命令直接删除，否则所有的线程都可以进行删除。

锁释放可以使用lua脚本保证原子性，先判断value是否是当前线程设置的，如果是就删除：

```lua
if redis.call('get',KEYS[1]) == ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end
```

# 单点Redis分布式锁的问题

- 必须设置一个过期时间，过期时间长短需要根据业务判断。
- 如果发生了故障转移，一个线程在master节点获取到了锁，但是没有同步到slave节点，master宕机，slave升级为master，其他线程获取分布式锁成功，就导致了有多个线程获取到了同一个锁。

# Redlock和Redisson

Redlock算法基于N个完全独立的Redis节点来实现，获取锁的操作步骤：

- 获取当前时间毫秒数
- 按顺序依次向N个Redis节点执行获取锁操作，使用相同的key和具有唯一性的value，也包含过期时间。获取锁的操作还需要有个超时时间，需要远小于key的过期时间。获取失败后应该立即尝试下一个Redis节点。
- 客户端使用当前时间减去开始获取锁的时间，得到了获取到锁使用的时间，如果客户端从大多数节点（N/2+1）成功获取到锁，并且总的获取锁的时间没有超过锁的有效期，才可以任务获取锁成功。
- 获取锁成功之后，锁的有效期等于最初的有效期减去上一步获取锁消耗的时间。
- 如果最终获取锁失败，客户端立刻向所有Redis节点发起释放锁的操作。

# zookeeper实现分布式锁

1. 先建立一个持久（PERSISTENT）节点，比如名字是：lock
2. 需要获取锁的时候，线程在lock节点下创建对应的临时顺序（EPHEMERAL_SEQUENTIAL）节点
3. 获取lock下所有的子节点，判断自己的节点是否是最小的节点，如果是最小的节点则获取锁成功
4. 如果当前不是最小节点，说明有其他线程获取到了锁，当前线程需要获取到前一个节点，并注册监听事件监听前一个节点
5. 当上一个节点完成后，释放锁，也就是删除了临时节点后，当前等待的节点会收到zk的通知事件，就可以获取到锁。

# Tair实现分布式锁

使用Tair的put方法来实现分布式锁，跟Redis的set类似，Tair可以传入版本号，cas保证只有一个能成功，同样还是使用key value加上过期时间来实现。

# 其他方式实现

mysql、文件实现分布式锁，mysql可以创建一个锁的表，mysql和文件的方式性能对比缓存来说不是很好。