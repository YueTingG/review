## 异步复制带来的问题

异步复制一个很容易想到的问题就是数据丢失，也就是**主服务器立马恢复客户端ok**，**这时候再慢慢把数据同步到从服务器**，但是加入这个过程中主服务器下线，从服务器就会丢失这个数据。

所以 redis 官网的说法就是如果部分数据一定要保证一致性，那么可以采用同步复制，其实就是调用 wait 命令

```
 WAIT  slave的数量  超时时间
```

https://www.knowledgedict.com/tutorial/redis-command-wait.html

## 脑裂

https://blog.csdn.net/LO_YUN/article/details/97131426

## 分布式锁

https://blog.csdn.net/j3T9Z7H/article/details/106699935

没有了，没有 setnx 和 setex 了，只有 set 命令了：http://redisdoc.com/string/set.html

https://www.cnblogs.com/zhili/p/redisdistributelock.html

#### 加锁

现在的锁都是**原子性**的了，可以同时 setnx + 超时时间。

```
SET key value [EX seconds|PX milliseconds] [NX|XX] [KEEPTTL]
```

**为什么要超时时间**？

怕线程挂了，待会没人去删除锁

#### 错误释放锁

就是锁因为超时时间已经到了，然后我去删除锁，这时候我有可能删错了别的进程的锁。**所以要在 value 中加入一个唯一标识 uuid**，来标识这个锁是不是我这个进程的。

如果已经释放了，但是现在上了锁，很有可能这个锁是别的进程的，而不是我的，我不能乱删。

```
String uuid = xxxx;
// 伪代码，具体实现看项目中用的连接工具
// 有的提供的方法名为 set 有的叫setIfAbsent
set Test uuid NX PX 3000
try{
// biz handle....
} finally {
    // unlock
    if(uuid.equals(redisTool.get('Test')){
        redisTool.del('Test');
    }
}

```

#### 释放锁

加锁的时候，set 命令是原子性的，但是解锁的时候，你要先获取锁，看看这把锁的值是不是自己的 uuid，然后再删。

这个过程有 get 获取锁，del 删除锁，但是这**两个步骤又不是原子性**，依然有**进程安全问题**。

具体场景有：客户端 A 加锁，一段时间之后客户端A进行解锁操作时，在执行 `redisTool.del()` 之前，锁突然过期了，此时客户端 B 尝试加锁成功，然后客户端A再执行 del 方法，则客户端 A 将客户端 B 的锁给解除了。 

解决方法就是 **lua 脚本**，同时在 lua 脚本中加入 get 和 del 命令，单吗如下：https://juejin.im/post/6844904137172189198#heading-4

```
String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";  

Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));
```

#### 总结

加锁

1. setnx
2. 过期时间

解锁

1. value 加唯一标识
2. 先 get 后 del，这两步要用 lua

## redis 与 session 的方案

#### redis 存储 session 的方案

首先，不要傻逼兮兮的觉得，这东西要什么方案，存就完事了，一般面试官问你这种问题，就是要问你具体的做法。

例如，存一个集群，还是一台 redis？使用 redis 什么数据结构？客户端存哪？

前面 2 个问题比较关键，后面大家都知道最后存 cookie。

https://jasonkayzk.github.io/2020/02/10/Redis%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8FSession/

#### 使用分布式 session 的好处（面试用的）

- 实现了 Session 共享；
- 可以水平扩展（增加 Redis 服务器）；
- 服务器重启 Session 不丢失（不过也要注意 Session 在 Redis 中的刷新/失效机制）；
- 不仅可以跨服务器 Session 共享，甚至可以跨平台（例如网页端和 APP 端）。

## 如何保证数据库和缓存的一致性问题

https://blog.csdn.net/u014401141/article/details/108507385

https://www.cnblogs.com/myseries/p/12068845.html 

https://blog.51cto.com/1991785/2129660 看这个

#### 操作

更新数据库，然后删除缓存

#### 为什么不是更新缓存，而是删除缓存

1. **更新缓存代价高昂**。其实很多时候我要更新某张表里面的字段，是需要多表一起查询得到的，然后再更新缓存，所以更新缓存的代价首先很多时候是高昂的。

2. **部分业务场景，写多查少**。
   举例，假如现在 1 分钟内，我频繁地修改表 20 次，然后只查询了 1 次，那么我如果选择更新缓存，那么我需要更新 20 次，而实际上大部分缓存刚更新缓就又被重新缓存，而只查询了 1 次。如果选择删除缓存，那么我更新多次，没有去更新缓存，等到查询的时候只更新一次，代价相对比较小。

#### 该方案一样会造成脏数据问题

1. 缓存刚好失效
2. 请求A查询数据库，得一个旧值
3. 请求B将新值写入数据库
4. 请求B删除缓存
5. 请求A将查到的旧值写入缓存 

#### 解决办法

1. 设置过期时间（要注意，这里使用缓存的情况都是按照不要求强一致性来要求的，如果项目对一致性要求极高，那么不应该使用缓存，应该使用 mysql，然后分表分库）
2. 暂时没看懂 https://blog.51cto.com/1991785/2129660