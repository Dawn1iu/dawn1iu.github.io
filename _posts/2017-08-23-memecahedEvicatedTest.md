---
    author: Dawn1iu
    date: 2017-08-23
    layout: post
    title: Memcached evicted 不同版本逻辑测试
    tags:
    - memcached
---

#### 测试目标
* 线上1.4.5版本是否直接从队尾踢数据 （是）
* 已到期数据是否计入被踢（不计入）  
* 新版本改进情况
* 确认新版本slab自动转换效果


#### 步骤
1. 存入一个对象，永久有效
2. 获取该对象，确定其处于LRU队尾
3. 定期（超过对象存活期）存入大量短存活期k/v 并get
4. 观察evictions情况

#### 1.4.25
1、线上1.4.5版本是否直接从队尾踢数据  
永久数据被踢，需关注下源码实现  
2、已到期数据是否计入被踢（不计入）  
不计入，区别于：  
evicted 释放对象数  
evicted_nonzero 设置了非0时间的LRU释放对象数  
reclaimed 使用过期对象空间存储对象次数


#### 1.4.5(online)



#### memcache 命令
start: memcached -m 8 -p 11211 -u ljq -d  
stats: stats / stats slabs / stats items  
dump: stats cachedump {slab_id} {limit num}   
get: get {session key}



