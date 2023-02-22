---
title: 缓存和存储一致性
date:  2023-02-22 22:41:18 +0800
category: 分布式
tags: 分布式
excerpt:
---

## 背景
一般更新缓存有有以下4种模式
1. 先更新Mysql，再更新Rdis
2. 先更新Rdis，再更新Mysql
3. 先更新Mysql，再删除Rdis
4. 先删除Rdis，再更新Mysql

> 先说结论：先更新Mysql，再删除Rdis的方案更好，如果对一致性要求很高可以引入消息队列异步删除，更优雅的方案可以使用订阅binlog方案

## 不一致问题
### 更新场景
#### 先更新Mysql，再更新Redis（反过来也一样）
![不一致1](/assets/img/distribute/cache1.png)
1. 请求A更新DB，k=v1
2. 请求B更新DB，k=v2
3. 请求B更新缓存，k=v2
4. 请求A更新缓存，k=v1   

> 此时DB中k=v2, cache中k=v1 不一致         
> 解决方案：加分布式锁

### 删除场景
#### 先删除Redis, 后更新Mysql
![不一致2](/assets/img/distribute/cache2.png)
1. 请求B删除缓存
2. 请求A读缓存cache miss
3. 请求A读数据库，k=v1
4. 请求B更新数据库，k=v2
5. 请求A写缓存k=v1   

> 此时DB中k=v2, cache中k=v1 不一致

#### 先更新Mysql, 后删除Redis
![不一致3](/assets/img/distribute/cache3.png)
1. 请求A读缓存（cache miss）
2. 请求A读数据库，k=v1
3. 请求B更新数据库，k=v2
4. 请求B删除缓存，此时缓存没有k，所以miss
5. 请求A写缓存，k=v1   

> 此时DB中k=v2, cache中k=v1 不一致         
> 理论上3、4步要比2、5步要慢，涉及到更新数据库，这个场景发生概率较低，所以优先选择先更新mysql再删除redis

## 最终一致性
1. 缓存设置过期时间，作为兜底方案；比如1分钟，不一致时间最多1分钟
2. 引入消息队列，异步删除缓存
![不一致4](/assets/img/distribute/cache4.png)
利用rocketMQ的事物消息，来让删除缓存的消息最终一定发送出去
3. MQ + 订阅binlog + 通用更新缓存服务
![不一致5](/assets/img/distribute/cache5.png)
使用Canel订阅Mysql binlog更新记录，投递到MQ，通用更新缓存服务消费消息，来统一更新缓存
