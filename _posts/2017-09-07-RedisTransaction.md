---
    author: Dawn1iu
    date: 2017-09-07
    layout: post
    title: Redis 事务
    tags:
    - redis
---
## Redis 事务
Redis 通过 MULTI、DISCARD、EXEC、WATCH 四个命令来实现事务功能，redis事务提供了一种“将多个命令打包， 然后一次性、按顺序地执行”的机制，并且事务在执行的期间不会主动中断 —— 服务器在执行完事务中的所有命令之后， 才会继续处理其他客户端的其他命令。

* MULTI 标记事务开始，将客户端的 REDIS_MULTI 选项打开， 让客户端从非事务状态切换到事务状态。
* EXEC 执行命令，服务器根据客户端所保存的事务队列， 以先进先出（FIFO）的方式执行事务队列中的命令。
* DISCARD 取消一个事务
* WATCH 只能在客户端进入事务状态之前执行，在事务状态下发送WATCH 命令会引发一个错误，但它不会造成整个事务失败，也不会修改事务队列中已有的数据。
* Redis 的事务是不可嵌套的， 当客户端已经处于事务状态， 而客户端又再向服务器发送 MULTI 时， 服务器只是简单地向客户端发送一个错误， 然后继续等待其他命令的入队。 MULTI 命令的发送不会造成整个事务失败， 也不会修改事务队列中已有的数据。

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set testk "testvalue"
QUEUED
127.0.0.1:6379> get testk
QUEUED
127.0.0.1:6379> sadd testk2 "v1" "v2" "v3"
QUEUED
127.0.0.1:6379> smembers testk2
QUEUED
127.0.0.1:6379> exec
1) OK
2) "testvalue"
3) (integer) 3
4) 1) "v2"
   2) "v3"
   3) "v1"
```

#### WATCH 实现
在每个代表数据库的 redis.h/redisDb 结构类型中， 都保存了一个 watched_keys 字典， 字典的键是这个数据库被监视的键， 而字典的值则是一个链表， 链表中保存了所有监视这个键的客户端。
![](http://redisbook.readthedocs.io/en/latest/_images/graphviz-9aea81f33da1373550c590eb0b7ca0c2b3d38366.svg)
![](http://redisbook.readthedocs.io/en/latest/_images/graphviz-fe5e31054c282a3cdd86656994fe1678a3d4f201.svg)
#### WATCH 触发
在任何对数据库键空间（key space）进行修改的命令成功执行之后 （比如 FLUSHDB 、 SET 、 DEL 、 LPUSH 、 SADD 、 ZREM ，诸如此类）， multi.c/touchWatchedKey 函数都会被调用 —— 它检查数据库的 watched_keys 字典， 看是否有客户端在监视已经被命令修改的键， 如果有的话， 程序将所有监视这个/这些被修改键的客户端的 REDIS_DIRTY_CAS 选项打开。当客户端发送 EXEC 命令、触发事务执行时， 服务器会对客户端的状态进行检查。

* watch一个自然过期的key不会触发事务失败

```
e.g.
```

## Redis 分布式锁实现
* SETNX：如果当前中没有值，则将其设置为并返回1，否则返回0。
* EXPIRE：将设置自动过期。
* GETSET：设值，并返回其原来的旧值。如果原来没有旧值，则返回nil。

简易伪代码：

```
boolean tryLock(){
	value = get(Key);
	if(value != null){
		return false;
	}
	
	
		if(setnx(key, uuid) == 1)	
			expire(time)
	

}


void unlock(){
	value = get(Key);
	if(value == uuid){
		del(key)
	}
}

```
问题：
1、当前ncr无法完美使用lua脚本，无法保证客户端命令一次性发送。
2、需要兼容到多个命令可能执行中断的问题，需要有一定的兼容方式。


通过储存过期时间防止设置过期时间失效

```
boolean tryLock(){
	if(setnx(key, now) == 1){
		true
	} 
	cacheTime = get(Key);
	if(cacheTime < now){
		old = getset(key, now)
		if(old == cacheTime){
			true;
		}
	}	
}


void unlock(){
	del(key)
}

```
问题：
删除时未做任何校验，可能存在一种情况：锁超时已被其他客户端占有后，原持有锁客户端进行unlock操作。


同时保存过期时间与uuid，getset可能导致缓存中uuid被一个未持有锁的对象
更新，导致真正持有锁的对象无法进行unlock




## Redis Lua语法
[官方说明](https://redis.io/commands/eval)

