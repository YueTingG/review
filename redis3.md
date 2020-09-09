## 异步复制带来的问题

异步复制一个很容易想到的问题就是数据丢失，也就是主服务器立马恢复客户端ok，这时候再慢慢把数据同步到从服务器，但是加入这个过程中主服务器下线，从服务器就会丢失这个数据。

所以 redis 官网的说法就是如果部分数据一定要保证一致性，那么可以采用同步复制，其实就是调用 wait 命令

```
 WAIT  slave的数量  超时时间
```

https://www.knowledgedict.com/tutorial/redis-command-wait.html

## 脑裂