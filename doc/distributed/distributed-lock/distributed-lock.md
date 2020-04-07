分布式锁一般主要分为两种实现方式：
```
    1.基于redis实现的分布式锁；
    2.基于zookeeper实现的分布式锁；
```

# 基于Redis实现的分布式锁

基于Redis实现的分布式锁一般分为两种：一种是`单机版`，一种是`集群版`。

## 单机版

从单个Redis节点上获取锁。部署机构包括：redis单实例/redis sentinel/redis cluster/redis主从。

### 实现方式

加锁命令：Redis命令

```
    SET resource_name my_random_value NX PX 30000
```

解锁命令：Lua脚本

```lua
    if redis.call("get",KEYS[1]) == ARGV[1] then
        return redis.call("del",KEYS[1])
    else
        return 0
    end
```

### 优点

加解锁指令都是原子操作，实现简单。

### 缺点

#### 1.单点问题

如果Redis服务器宕机了，整个服务都将不可用。

#### 2.主从切换问题

由于redis是异步同步数据，所以在集群模式下（哨兵或集群），如果主节点还未将写操作同步给从节点就宕机了，那么就会丢失写操作数据，锁也就释放了，当主备切换后，其它客户端可以获取到同一把锁，这样锁就不安全了。

#### 3.锁超时问题

由于各种原因，比如：GC的STW，长时间的网络I/O等原因，导致客户端业务逻辑还未执行完成，锁就已经过期释放了，那么其他客户端可能会同时拿到锁，这时锁的安全性就被破坏了。

## 集群版

从多个redis节点上获取锁，比如基于Redlock算法实现的分布式锁。这里的集群不是指redis集群或者redis哨兵，而是节点之间相互孤立，节点间不存在主从复制和协作。

### 实现算法

```
Redlock算法：

前提条件：假设有N个redis节点，N为大于2的奇数，节点之间相互孤立，节点间不存在主从复制和协作。
基于Redlock算法实现分布式锁的步骤：
（1）获取当前系统时间；
（2）使用相同的key和随机数逐个从每个节点上获取锁；
（3）判断在锁过期时间之内（（1）中获取到的当前系统时间+锁过期时间>获取到足够多锁的时间）是否获取到了足够多的锁（大于（N/2+1）个）；
（4）如果在锁过期之前获取到足够多的锁，则说明锁获取成功；
（5）如果锁获取失败，则需要在每个节点上面释放锁；
```
### 实现方式

采用Redisson类库，Redisson类库中基于Redlock算法实现了分布式锁。

### 优点

1. 可以解决单点问题。就算有N/2节点同时宕机了，集群还是继续工作。
2. 可以解决主从切换问题。当一个节点宕机后，这时不会有其他从节点顶替原来的主节点。所以，只要前一个客户端没有释放锁，那么其他客户端获取不到同一把锁。

### 缺点

#### 1.锁超时问题

由于各种原因，比如：GC的STW，长时间的网络I/O等原因，导致客户端业务逻辑还未执行完成，锁就已经过期释放了，那么其他客户端可能会同时拿到锁，这时锁的安全性就被破坏了。

## Redsson类库

Redsson实现了单机版的分布式锁，支持多种部署架构（redis单实例/redis sentinel/redis cluster/redis主从），也基于Redlock算法实现了集群版的分布式锁。

## redis分布式锁存在的问题

### 1.锁超时问题

由于各种原因，比如：GC的STW，长时间的网络I/O等原因，导致客户端业务逻辑还未执行完成，锁就已经过期释放了，那么其他客户端可能会同时拿到锁，这时锁的安全性就被破坏了。


# 基于Zookeeper实现的分布式锁
zookeeper数据结构类似于文件系统的目录结构，资源目录对应锁，目录下面的文件节点就是需要获取锁的客户端。
zookeeper集群数据采用强一致性

# 参考资料

1. 【官方文档-英文】Distributed locks with Redis - [https://redis.io/topics/distlock](https://redis.io/topics/distlock)
2. 【官方文档-中文】Redis分布式锁 - [http://www.redis.cn/topics/distlock.html](http://www.redis.cn/topics/distlock.html)
3. 再有人问你分布式锁，这篇文章扔给他 - [https://juejin.im/post/5bbb0d8df265da0abd3533a5#heading-0](https://juejin.im/post/5bbb0d8df265da0abd3533a5#heading-0)
4. 拜托，面试请不要再问我Redis分布式锁的实现原理 - [http://www.imooc.com/article/284859](http://www.imooc.com/article/284859)