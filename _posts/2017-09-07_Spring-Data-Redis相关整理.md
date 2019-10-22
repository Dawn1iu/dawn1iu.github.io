---
    author: Dawn1iu
    date: 2017-09-07
    layout: post
    title: Spring Redis相关整理
    categories:
    - nosql
    tags:
    - redis
    - nosql
---
## Spring Redis相关整理

###  楔子

希望整理自己在使用redisTemplate过程中遇到的问题和坑，明确RedisTemlate的使用逻辑和规范，通过redisTemplate的事务实现方式复习Spring的事务管理逻辑。

###  1. 总
spring-data-redis提供了redis操作的封装和实现；RedisTemplate模板类封装了redis连接池管理的逻辑，业务代码无须关心获取，释放连接逻辑；spring redis同时支持了Jedis，Jredis,rjc 客户端操作；
 
spring redis 设计优点可以分为以下几个方面：

1. Redis连接管理：用adapter的方式封装了Jedis，Jredis，Rjc等不同redis客户端连接，用抽象工厂的方式供依赖注入。
2. Redis操作封装：value，list，set，sortset，hash划分为不同操作
3. Redis序列化：能够以插件的形式配置想要的序列化实现
4. Redis操作模板化：redis操作过程分为：获取连接，业务操作，释放连接；模板方法使得业务代码只需要关心业务操作
5. Redis事务模块：通过sessioncallback实现，集成spring的transaction manager。

![spring redis类图](http://dl2.iteye.com/upload/attachment/0078/2550/65141958-9e52-3751-87e2-73722d1184ac.jpg)

####  1.1 连接管理
| 类名       | 职责         
| -------------|-------------| 
| RedisCommands | 继承了Redis各种数据类型操作的整合接口|
| RedisConnection |  抽象了不同底层redis客户端类型：不同类型的redis客户端可以创建不同实现     |   
| RedisConnectionFactory | 抽象Redis连接工厂，不同类型的redis客户端实现不同的工厂|  
| JedisConnection | 实现RedisConnection接口，将操作委托给Jedis      |  
| JedisConnectionFactory | 实现RedisConnectionFactory接口，创建JedisConnection      |  

####  1.2 操作封装
#####  1.2.1 RedisTemplate
RedisOperations接口的实现类就是RedisTemplate本身，主要提供了一些对Redis键，事务，运行脚本等命令的支持，不负责数据的读写。真正的操作方法也是通过CallBack机制调用的connection本身的api实现。

#####  1.2.1 ValueOperationsEditor
先看下ValueOperations这个接口的实现类为DefaultValueOperations，default这个类同时继承AbstractOperation

```
	//非公开的，需要传入template来构造
		
	DefaultValueOperations(RedisTemplate<K, V> template) {
		super(template);
	}
```
在RedisTemplate中，已经提供了一个方法`opsForValue()`,这个方法会返回一个默认的操作类。

同时，我们可以直接通过redisTemplate注入一个ValueOperations。

```
    @Resource(name = "redisTemplate")
    private ValueOperations<String, Object> vOps;
```

###  2. 通过RedisTemplate和RedisConnection看代码设计

#### 2.1 总览
RedisTemplate提供了对连接操作的模板化支持；采用RedisCallback来回调业务操作，使得业务代码无需关心连接处理，以及其他异常处理等过程，简化redis操作；
 
RedisTemplate继承RedisAccessor 类，配置管理RedisConnectionFactory实现；使得RedisTemplate无需关心底层redis客户端类型
 
RedisTemplate实现RedisOperations接口，提供value，list，set，sortset，hash以及其他redis操作方法；value，list，set，sortset，hash等操作划分为不同操作类：ValueOperations，ListOperations，SetOperations，ZSetOperations，HashOperations以及bound接口；这些操作都提供了默认实现，这些操作都采用RedisCallback回调实现相关操作
 
RedisTemplate组合了多个不同RedisSerializer示例，以实现对于key，value的序列化支持；可以方便地实现自己的序列化工具；

//todo

###  3. 通过sessioncallback看事务实现
####  3.1 通过rename看sessionCallback实现
实现并没有使用内部持有的RedisOperations，而是CollectionUtils.rename通过事务来保证进行rename操作时，原key一定存在。

在调用 SessionCallback 的实现进行具体操作前后，对连接进行了绑定和解绑。然后在session.execute中，会调用operations.watch(key)来监控值的变化。
 
连接绑定和解绑也是通过 RedisConnectionUtils 完成的，里面是通过TransactionSynchronizationManager将连接绑定到当前线程的。

//todo
 
###  4.坑

####  4.1 序列化与incr问题

incr并未进行value的序列化操作，jedis底层使用了`String.valueOf()`,所以incr与get/set等操作配合时只能使用String的值序列化方式


####  4.2 twemproxy不支持事务问题

线上环境使用session


####  4.3 自定义RedisCallback序列化问题
实现rediscallback的时候需要注意自己做序列化和反序列化。

对于单Key的操作可以模仿ValueDeserializingRedisCallback通过继承RedisCallback来在模板中实现序列化与反序列化操作，但是对于多个key或更复杂的操作，是否有好办法解决？

####  4.4 使用pipline模式的返回问题

pipline模式下操作返回值其实是在closePipline时返回，但redisTemplate本身不处理。











