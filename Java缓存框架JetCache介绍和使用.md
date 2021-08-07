---
title: Java缓存框架JetCache介绍和使用
date: 2021-08-07 01:15:24
tags: JetCache
---

# 一、简介

JetCache是一个基于Java的缓存系统封装，提供统一的API和注解来简化缓存的使用。 JetCache提供了比SpringCache更加强大的注解，可以原生的支持TTL、两级缓存、分布式自动刷新，还提供了`Cache`接口用于手工缓存操作。<!-- more --> 

当前有四个实现，`RedisCache`、`TairCache`、`CaffeineCache`和一个简易的`LinkedHashMapCache`，要添加新的实现也是非常简单的。

全部特性:

- 通过统一的API访问Cache系统

- 通过注解实现声明式的方法缓存，支持TTL和两级缓存

- 通过注解创建并配置`Cache`实例

- 针对所有`Cache`实例和方法缓存的自动统计

- Key的生成策略和Value的序列化策略是可以配置的

- 分布式缓存自动刷新，分布式锁 (2.2+)

- 异步Cache API (2.2+，使用Redis的lettuce客户端时)

- Spring Boot支持

# 二、环境要求

JetCache需要JDK1.8、Spring Framework4.0.8以上版本。

Spring Boot为可选，需要1.1.9以上版本。

如果不使用注解（仅使用jetcache-core），Spring Framework也是可选的。

# 三、方法缓存

JetCache方法缓存原生提供了TTL支持，以保证最终一致，并且支持二级缓存。JetCache2.4以后支持基于注解的缓存更新和删除。

@Cached注解可以为一个方法添加缓存，@CacheUpdate用于更新缓存，@CacheInvalidate用于移除缓存元素。注解可以加在接口上也可以加在类上，加注解的类必须是一个spring bean

```java
public interface UserService {

    @Cached(name="userCache.", key="#userId", expire = 3600)
    User getUserById(long userId);

    @CacheUpdate(name="userCache.", key="#user.userId", value="#user")
    void updateUser(User user);

    @CacheInvalidate(name="userCache.", key="#userId")
    void deleteUser(long userId);

}
```

key使用Spring的SpEL脚本来指定。比如这里的`key="#userId"`,`key="#user.userId"`

## @Cached

| 属性           | 默认值           | 说明                                                         |
| -------------- | ---------------- | ------------------------------------------------------------ |
| area           | “default”        | 如果在配置中配置了多个缓存area，在这里指定使用哪个area       |
| name           | 未定义           | 指定缓存的唯一名称，不是必须的，如果没有指定，会使用类名+方法名。name会被用于远程缓存的key前缀。另外在统计中，一个简短有意义的名字会提高可读性。 |
| key            | 未定义           | 使用SpEL指定key，如果没有指定会根据所有参数自动生成。        |
| expire         | 未定义           | 超时时间。如果注解上没有定义，会使用全局配置，如果此时全局配置也没有定义，则为无穷大 |
| timeUnit       | TimeUnit.SECONDS | 指定expire的单位                                             |
| cacheType      | CacheType.REMOTE | 缓存的类型，包括CacheType.REMOTE、CacheType.LOCAL、CacheType.BOTH。如果定义为BOTH，会使用LOCAL和REMOTE组合成两级缓存 |
| localLimit     | 未定义           | 如果cacheType为LOCAL或BOTH，这个参数指定本地缓存的最大元素数量，以控制内存占用。如果注解上没有定义，会使用全局配置，如果此时全局配置也没有定义，则为100 |
| localExpire    | 未定义           | 仅当cacheType为BOTH时适用，为内存中的Cache指定一个不一样的超时时间，通常应该小于expire |
| serialPolicy   | 未定义           | 指定远程缓存的序列化方式。可选值为SerialPolicy.JAVA和SerialPolicy.KRYO。如果注解上没有定义，会使用全局配置，如果此时全局配置也没有定义，则为SerialPolicy.JAVA |
| keyConvertor   | 未定义           | 指定KEY的转换方式，用于将复杂的KEY类型转换为缓存实现可以接受的类型，当前支持KeyConvertor.FASTJSON和KeyConvertor.NONE。NONE表示不转换，FASTJSON可以将复杂对象KEY转换成String。如果注解上没有定义，会使用全局配置。 |
| enabled        | TRUE             | 是否激活缓存。例如某个dao方法上加缓存注解，由于某些调用场景下不能有缓存，所以可以设置enabled为false，正常调用不会使用缓存，在需要的地方可使用CacheContext.enableCache在回调中激活缓存，缓存激活的标记在ThreadLocal上，该标记被设置后，所有enable=false的缓存都被激活 |
| cacheNullValue | FALSE            | 当方法返回值为null的时候是否要缓存                           |
| condition      | 未定义           | 使用SpEL指定条件，如果表达式返回true的时候才去缓存中查询     |
| postCondition  | 未定义           | 使用SpEL指定条件，如果表达式返回true的时候才更新缓存，该评估在方法执行后进行，因此可以访问到#result |

## @CacheUpdate

| 属性      | 默认值    | 说明                                                         |
| --------- | --------- | ------------------------------------------------------------ |
| area      | “default” | 如果在配置中配置了多个缓存area，在这里指定使用哪个area，指向对应的@Cached定义。 |
| name      | 未定义    | 指定缓存的唯一名称，指向对应的@Cached定义。                  |
| key       | 未定义    | 使用SpEL指定key                                              |
| value     | 未定义    | 使用SpEL指定value                                            |
| condition | 未定义    | 使用SpEL指定条件，如果表达式返回true才执行更新，可访问方法结果#result |

## @CacheInvalidate

| 属性      | 默认值    | 说明                                                         |
| --------- | --------- | ------------------------------------------------------------ |
| area      | “default” | 如果在配置中配置了多个缓存area，在这里指定使用哪个area，指向对应的@Cached定义。 |
| name      | 未定义    | 指定缓存的唯一名称，指向对应的@Cached定义。                  |
| key       | 未定义    | 使用SpEL指定key                                              |
| condition | 未定义    | 使用SpEL指定条件，如果表达式返回true才执行删除，可访问方法结果#result |

使用@CacheUpdate和@CacheInvalidate的时候，相关的缓存操作可能会失败（比如网络IO错误），所以指定缓存的超时时间是非常重要的。

# 四、创建缓存实例

通过@CreateCache注解创建一个缓存实例，默认超时时间是100秒

```java
@CreateCache(expire = 100)
private Cache<Long, UserDO> userCache;
```

用起来就像map一样

```java
UserDO user = userCache.get(123L);
userCache.put(123L, user);
userCache.remove(123L);
```

创建一个两级（内存+远程）的缓存，内存中的元素个数限制在50个。

```java
@CreateCache(name = "UserService.userCache", expire = 100, cacheType = CacheType.BOTH, localLimit = 50)
private Cache<Long, UserDO> userCache;
```

## @CreateCache

| 属性         | 默认值           | 说明                                                         |
| ------------ | ---------------- | ------------------------------------------------------------ |
| area         | “default”        | 如果需要连接多个缓存系统，可在配置多个cache area，这个属性指定要使用的那个area的name |
| name         | 未定义           | 指定缓存的名称，不是必须的，如果没有指定，会使用类名+方法名。name会被用于远程缓存的key前缀。另外在统计中，一个简短有意义的名字会提高可读性。如果两个@CreateCache的name和area相同，它们会指向同一个Cache实例 |
| expire       | 未定义           | 该Cache实例的默认超时时间定义，注解上没有定义的时候会使用全局配置，如果此时全局配置也没有定义，则取无穷大 |
| timeUnit     | TimeUnit.SECONDS | 指定expire的单位                                             |
| cacheType    | CacheType.REMOTE | 缓存的类型，包括CacheType.REMOTE、CacheType.LOCAL、CacheType.BOTH。如果定义为BOTH，会使用LOCAL和REMOTE组合成两级缓存 |
| localLimit   | 未定义           | 如果cacheType为CacheType.LOCAL或CacheType.BOTH，这个参数指定本地缓存的最大元素数量，以控制内存占用。注解上没有定义的时候会使用全局配置，如果此时全局配置也没有定义，则取100 |
| serialPolicy | 未定义           | 如果cacheType为CacheType.REMOTE或CacheType.BOTH，指定远程缓存的序列化方式。JetCache内置的可选值为SerialPolicy.JAVA和SerialPolicy.KRYO。注解上没有定义的时候会使用全局配置，如果此时全局配置也没有定义，则取SerialPolicy.JAVA |
| keyConvertor | 未定义           | 指定KEY的转换方式，用于将复杂的KEY类型转换为缓存实现可以接受的类型，JetCache内置的可选值为KeyConvertor.FASTJSON和KeyConvertor.NONE。NONE表示不转换，FASTJSON通过fastjson将复杂对象KEY转换成String。如果注解上没有定义，则使用全局配置。 |

# 五、配置详解

配置文件案例

```yaml
jetcache:
  statIntervalMinutes: 15
  areaInCacheName: false
  hidePackages: com.alibaba
  local:
    default:
      type: caffeine
      limit: 100
      keyConvertor: fastjson
      expireAfterWriteInMillis: 100000
    otherArea:
      type: linkedhashmap
      limit: 100
      keyConvertor: none
      expireAfterWriteInMillis: 100000
  remote:
    default:
      type: redis
      keyConvertor: fastjson
      valueEncoder: java
      valueDecoder: java
      poolConfig:
        minIdle: 5
        maxIdle: 20
        maxTotal: 50
      host: ${redis.host}
      port: ${redis.port}
    otherArea:
      type: redis
      keyConvertor: fastjson
      valueEncoder: kryo
      valueDecoder: kryo
      poolConfig:
        minIdle: 5
        maxIdle: 20
        maxTotal: 50
      host: ${redis.host}
      port: ${redis.port}
```

| 属性                                                      | 默认值 | 说明                                                         |
| --------------------------------------------------------- | ------ | ------------------------------------------------------------ |
| jetcache.statIntervalMinutes                              | 0      | 统计间隔，0表示不统计                                        |
| jetcache.areaInCacheName                                  | TRUE   | jetcache-anno把cacheName作为远程缓存key前缀，2.4.3以前的版本总是把areaName加在cacheName中，因此areaName也出现在key前缀中。2.4.4以后可以配置，为了保持远程key兼容默认值为true，但是新项目的话false更合理些。 |
| jetcache.hiddenPackages                                   | 无     | @Cached和@CreateCache自动生成name的时候，为了不让name太长，hiddenPackages指定的包名前缀被截掉 |
| jetcache.[local\|remote].${area}.type                     | 无     | 缓存类型。tair、redis为当前支持的远程缓存；linkedhashmap、caffeine为当前支持的本地缓存类型 |
| jetcache.[local\|remote].${area}.keyConvertor             | 无     | key转换器的全局配置，当前只有一个已经实现的keyConvertor：fastjson。仅当使用@CreateCache且缓存类型为LOCAL时可以指定为none，此时通过equals方法来识别key。方法缓存必须指定keyConvertor |
| jetcache.[local\|remote].${area}.valueEncoder             | java   | 序列化器的全局配置。仅remote类型的缓存需要指定，可选java和kryo |
| jetcache.[local\|remote].${area}.valueDecoder             | java   | 序列化器的全局配置。仅remote类型的缓存需要指定，可选java和kryo |
| jetcache.[local\|remote].${area}.limit                    | 100    | 每个缓存实例的最大元素的全局配置，仅local类型的缓存需要指定。注意是每个缓存实例的限制，而不是全部，比如这里指定100，然后用@CreateCache创建了两个缓存实例（并且注解上没有设置localLimit属性），那么每个缓存实例的限制都是100 |
| jetcache.[local\|remote].${area}.expireAfterWriteInMillis | 无穷大 | 以毫秒为单位指定超时时间的全局配置(以前为defaultExpireInMillis) |
| jetcache.local.${area}.expireAfterAccessInMillis          | 0      | 需要jetcache2.2以上，以毫秒为单位，指定多长时间没有访问，就让缓存失效，当前只有本地缓存支持。0表示不使用这个功能。 |

${area}对应@Cached和@CreateCache的area属性。注意如果注解上没有指定area，默认是"default"。

# 六、快速上手

## POM

```xml
<dependency>
    <groupId>com.alicp.jetcache</groupId>
    <artifactId>jetcache-starter-redis</artifactId>
    <version>2.5.14</version>
</dependency>
```

## application.yml

```yaml
jetcache:
  statIntervalMinutes: 15
  areaInCacheName: false
  local:
    default:
      type: linkedhashmap
      keyConvertor: fastjson
  remote:
    default:
      type: redis
      keyConvertor: fastjson
      valueEncoder: java
      valueDecoder: java
      poolConfig:
        minIdle: 5
        maxIdle: 20
        maxTotal: 50
      host: 127.0.0.1
      port: 6379
      password：123456
```

## 启动类

@EnableMethodCache，@EnableCreateCacheAnnotation这两个注解分别激活@Cached和@CreateCache注解

```java
@SpringBootApplication
@EnableMethodCache(basePackages = "com.company.mypackage")
@EnableCreateCacheAnnotation
public class MySpringBootApp {

    public static void main(String[] args) {
        SpringApplication.run(MySpringBootApp.class);
    }

}
```

# 七、官方详细文档

https://github.com/alibaba/jetcache/wiki/Home_CN

