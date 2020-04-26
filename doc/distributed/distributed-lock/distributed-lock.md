分布式锁一般主流实现方式有两种：
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

zookeeper的数据结构类似于文件系统，是一种树形结构。

```
    基于zookeeper实现分布式锁过程大致如下：
    
    客户端A从zookeeper中获取锁时，会在zookeeper数据模型中创建一个znode作为锁对象，
    然后还会在该锁znode节点下创建一个子znode作为客户端A对象（这类子节点名称具有顺序性，每次新增的子节点名称都会在上一个子节点的基础上加1）。
    当另一个客户端B来获取同一把锁时，同样会在锁znode节点下创建一个子节点，
    同时客户端会判断锁节点下是否有子节点，如果没有，那么客户端B直接获取到锁，
    如果有，那么客户端B就会创建一个Watcher监视上一个字节点的删除事件，
    一旦上一个子节点客户端A被删除触发删除事件，那么就会通知客户端B去获取锁。
```

## 实现方式

Curator是基于Zookeeper API实现的Java客户端，它可以简化我们对Zookeeper进行操作。
它还封装了分布式锁功能，这样我们就不用再自己实现了。

## 优点

由于zookeeper集群基于Zab协议实现了最终一致性，所以任何时候任一节点上的数据都是一样的。
由于zookeeper server与zookeeper client使用了Session保持会话，客户端会定时发送心跳给服务端保持会话，
当客户端宕机了，那么，对应zookeeper上的锁会释放掉。
当服务端Leader宕机了，那么，zookeeper通过故障恢复会选举出新的Leader，并把它上面的数据同步给其他Follower。
只要zookeeper集群同时宕机数量不超过一半，那么整个集群就依然可用，解决单点问题。
Leader故障恢复过程中zookeeper集群是不用的。



# 参考资料

1. 【官方文档-英文】Distributed locks with Redis - [https://redis.io/topics/distlock](https://redis.io/topics/distlock)
2. 【官方文档-中文】Redis分布式锁 - [http://www.redis.cn/topics/distlock.html](http://www.redis.cn/topics/distlock.html)
3. 再有人问你分布式锁，这篇文章扔给他 - [https://juejin.im/post/5bbb0d8df265da0abd3533a5#heading-0](https://juejin.im/post/5bbb0d8df265da0abd3533a5#heading-0)
4. 拜托，面试请不要再问我Redis分布式锁的实现原理 - [http://www.imooc.com/article/284859](http://www.imooc.com/article/284859)
5. Zookeeper——一致性协议:Zab协议 - [https://www.jianshu.com/p/2bceacd60b8a](https://www.jianshu.com/p/2bceacd60b8a)
6. Zookeeper 教程 - [https://www.w3cschool.cn/zookeeper/](https://www.w3cschool.cn/zookeeper/)
7. 学习Redisson的不二之选 - [https://github.com/redisson/redisson/wiki/%E7%9B%AE%E5%BD%95](https://github.com/redisson/redisson/wiki/%E7%9B%AE%E5%BD%95)
8. Redisson实现Redis分布式锁的N种姿势 - [https://www.jianshu.com/p/f302aa345ca8](https://www.jianshu.com/p/f302aa345ca8)
9. Zookeeper到底是干嘛的 - [https://www.cnblogs.com/ultranms/p/9585191.html](https://www.cnblogs.com/ultranms/p/9585191.html)