---
title: Redisson实现分布式锁
categories: Java
date: 2021-08-07 22:18:25
tags: 分布式锁
---

# Redisson项目介绍

Redisson是架设在Redis基础上的一个Java驻内存数据网格。充分的利用了Redis键值数据库提供的一系列优势，基于Java实用工具包中常用接口，为使用者提供了一系列具有分布式特性的常用工具类。使得原本作为协调单机多线程并发程序的工具包获得了协调分布式多机多线程并发系统的能力，大大降低了设计和研发大规模分布式系统的难度。同时结合各富特色的分布式服务，更进一步简化了分布式环境中程序相互之间的协作。<!-- more -->

https://github.com/redisson/redisson/wiki/Redisson%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D

# 引入redisson

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.13.1</version>
</dependency>
```

# 配置RedissonClient

```java
@Bean
public RedissonClient redissonClient() {
   Config config = new Config();
   String address = "redis://" + properties.getHost() + ":" + properties.getPort();
   config.useSingleServer()
         .setAddress(address)
         .setPassword(properties.getPassword());
   RedissonClient redisson = Redisson.create(config);
   return redisson;
}
```

# 使用锁

```java
@Autowired
private RedissonClient redissonClient;

// 获取锁
RLock lock = redissonClient.getLock(lockKey);
if(!lock.tryLock(expire_time,TimeUnit.Seconds)){
   return "lock fail"
}
// lock successful
try{
  // 业务代码
} catch() {
} finally {
  // 释放锁
  lock.unlock();
}
```

# 配置属性

```properties
spring.redis.host = 127.0.0.1
spring.redis.port = 6379
spring.redis.password = 
```

Redisson Gibhub地址-> https://github.com/redisson/redisson

