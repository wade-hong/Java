# 基于Redis实现的分布式锁

基于Redis实现的分布式锁一般分为两种情况：一种是**`单机版`**，一种是**`集群版`**。

## 单机版

单个Redis实例。

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

#### 2.超时问题
如果客户端A获取锁之后还没来得及释放锁，锁就过期了，这时其他客户端就可以同时获取到这把锁，这样就会出现多个客户端同时获取到同一把锁的安全问题。

## 集群版

多个redis节点，节点间是孤立的，节点间不存在主从复制和协作。

### 实现算法

```
Redlock算法：

前提条件：假设有N个redis节点，N为大于2的奇数，节点之间相互孤立，节点间不存在主从复制和协作。

基于Redlock算法实现分布式锁的步骤：
（1）获取当前系统时间；
（2）使用相同的key和随机数逐个从每个节点上获取锁；
（3）判断在锁过期时间之内（（1）中获取到的当前系统时间+锁过期时间>获取到足够多锁的时间）是否获取到了足够多的锁（大于（N/2-1）个）；
（4）如果在锁过期之前获取到足够多的锁，则说明锁获取成功；
（5）如果锁获取失败，则需要在每个节点上面释放锁；
```
### 实现方式

Redisson里面基于Redlock算法实现了分布式锁，是一种Java实现方式。

# 基于Zookeeper实现的分布式锁


# 推荐阅读

1.【官方文档-英文】Distributed locks with Redis - [https://redis.io/topics/distlock](https://redis.io/topics/distlock)
2.【官方文档-中文】Redis分布式锁 - [http://www.redis.cn/topics/distlock.html](http://www.redis.cn/topics/distlock.html)

# 参考资料

1.再有人问你分布式锁，这篇文章扔给他 - [https://juejin.im/post/5bbb0d8df265da0abd3533a5#heading-0](https://juejin.im/post/5bbb0d8df265da0abd3533a5#heading-0)