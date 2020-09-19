## 异步复制带来的问题

异步复制一个很容易想到的问题就是数据丢失，也就是**主服务器立马恢复客户端ok**，**这时候再慢慢把数据同步到从服务器**，但是加入这个过程中主服务器下线，从服务器就会丢失这个数据。

所以 redis 官网的说法就是如果部分数据一定要保证一致性，那么可以采用同步复制，其实就是调用 wait 命令

```
 WAIT  slave的数量  超时时间
```

https://www.knowledgedict.com/tutorial/redis-command-wait.html

## 脑裂

## 分布式锁

https://blog.csdn.net/j3T9Z7H/article/details/106699935

没有了，没有 setnx 和 setex 了，只有 set 命令了：http://redisdoc.com/string/set.html

#### 加锁

现在的锁都是**原子性**的了，可以同时 setnx+超时时间。

```
SET key value [EX seconds|PX milliseconds] [NX|XX] [KEEPTTL]
```

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

解决方法就是 **lua 脚本**，同时在 lua 脚本中加入 get 和 del 命令，单吗如下：https://juejin.im/post/6844904137172189198#heading-4

```
String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";  

Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));
```

