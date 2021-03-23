---
typora-root-url: ./
---

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

## redis 事务

#### 开启事务的命令

```
redis > mutil // 此命令相当于 mysql 的 begin
ok

redis> set "name" "xiaoming" // 存入命令
queued

redis> get "name" // 存入命令
queued

reids> exec // 遍历队列,然后执行命令
1) ok
2) "xiaoming"
```

#### 大体过程

1. 事务开始
2. 命令入队
3. 命令执行

#### 事务开始

执行 `mutil` 后，事务标志开始，客户端有个 flag，会把它置成开启事务的状态，总之就是开启一个标志。

#### 命令入队

redis 内部有个事务队列，会把 mutil 后写入的命令放到队列里面，暂时存着，不执行。

#### 执行事务

执行 exec 后，遍历事务队列，然后依次执行命令。

**注意**：redis 执行事务是有条件的，一个条件是命令必须**语法正确**，现在版本是，如果入队的命令有一个语法错误，那么 exec 不会执行任何命令，连正确的也**不会执行**（原子嘛）。

**注意**：执行的另外一个条件是 watch 命令监听的 key 必须未曾修改。

**注意**：上面说的，错误是指语法错误，也就是说，redis 是先能进行检查，如果是运行错误的呢？redis 无法进行检查，它还是会执行命令的，只是说在执行的错误给你报错。

## redis 的 watch 命令

#### 命令

```
redis> watch "name"
ok
```

这就监听了某个 key。

#### 数据结构

redis 数据库里面存放着字典，里面记录了哪些客户端监听了哪些 key，字典里面 key 是你监听的 key，value 是一条链表，里面的每一个结点是一个个客户端。

![](/graphviz-9aea81f33da1373550c590eb0b7ca0c2b3d38366.svg)

#### 实现

具体实现的逻辑是基于它的数据结构本身的，当客户端执行了诸如，set、push、sadd 之类修改有关的命令的时候，redis 会对字典进行检查，看看是否有别的客户端刚才监听了这个 key，如果有那么待会 exec 就不能成功。

所以**触发的时机**：就是当客户端执行一些跟修改有关的命令的时候，会进行检查。

## redis 的命令执行

https://www.jianshu.com/p/361cb9cd13d5

## redis 事务的 acid 特性

#### 原子性

首先，redis 的事务符合原子性，因为里面有个事务队列，队列里面的命令要么全部成功，要么全部失败。

**redis 不支持回滚**。

（这点你要跟 mysql 进行区分，mysql 在输入 sql 的时候，实际上 sql 是执行的了，只不过数据的反应是在 undo 日志里面，所以才有回滚，毕竟你执行了，你要还原肯定要回滚，redis 呢？你压根都没执行，不用回滚，redis 的逻辑就是最后我检查一下，要不要执行事务队列的命令，而不是当你输入命令的时候就执行命令，所以 redis 才没有回滚的说法。）

#### 一致性

一致性的定义是事务执行前后，redis 仍然是一致的。

这里的一致性主要就是指仍然满足数据本身的定义，然后不能出现非法的数据。

为什么 redis 能满足一致性呢？主要是它能检查一些错误，然后保证这些错误不影响 redis。

##### 入队错误（语法检查）

当你在命令入队的时候，入队了一个语法错误的命令或者完全不存在的命令，此后如果执行 exec，全部命令都不会执行（保证错误的命令不会执行）

```
redis 127.0.0.1:6379> MULTI
OK

redis 127.0.0.1:6379> set key // 语法错误，或者这条语法完全不存在，都不会执行
(error) ERR wrong number of arguments for 'set' command

redis 127.0.0.1:6379> EXISTS key
QUEUED

redis 127.0.0.1:6379> EXEC
(error) EXECABORT Transaction discarded because of previous errors.
```

##### 执行错误（入队的时候发现不了这个错误）

执行的时候，如果某一条命令的语法本身并没有问题，但是执行之后确实错误的，例如 sadd key val，但是 key 的类型确实 string，那么它不应该是 sadd，而应该是 set。但是这个在语法检查的时候却是发现不了的。

```
redis＞MULTI
OK
redis＞SET key 1
QUEUED
redis＞SADD key 2 // key 是 string，却用了 sadd
QUEUED
redis＞SET key 3
QUEUED
redis＞EXEC
1) OK
2) (error) ERR Operation against a key holding the wrong kind of value
3) OK
redis＞GET key
"3"
```

##### 服务器停机

redis 拥有 rdb 和 aof 能保证重启之后，redis 的数据是一致性的。

#### 隔离性

首先，要明白隔离性往往被破坏的原因是什么？就是因为数据库本身具有并发的性能，并发执行 sql 或者命令的时候，互相影响，但是你要执行 redis 执行命令的时候是单线程，根本不存在线程安全问题，所以 redis 具有隔离性。

**redis 执行命令是单线程的**，所以具有隔离性。

#### 持久性

redis 的事务本身只不过是包裹了一组命令，所以 redis 事务本身的持久性就是指 redis 本身的持久性，而我们知道 redis 本身具备持久性，依赖的是 rdb 和 aof。