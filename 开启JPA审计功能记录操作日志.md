---
title: 开启JPA审计功能记录操作日志
categories: Java
date: 2021-08-08 18:28:06
tags: JPA
---

# 数据库审计

数据库审计是指当数据库有记录变更时，可以记录数据库的变更时间和变更人等，这样以后出问题回溯问责也比较方便。

`Spring Data JPA`为我们提供了方便的`Audit`功能<!-- more -->

# 注解字段

主要有 下面几核心的基础注解

`@CreateBy` 表示该字段为 “创建者” 字段，在实体被 insert 之时设置

`@LastModifiedBy` 表示该字段为 “最后修改者” 字段，在实体被 update 之时设置

`@CreatedDate` 表示该字段为 “创建时间” 字段，在实体被 insert 之时设置

`@LastmodifiedDate` 表示该字段为 “修改时间” 字段，在实体被 update 之时设置

`@EntityListeners` 作用于实体类上，用来监听实体类的变化

# 核心接口

`PostLoadEventListener`	

将实体加载到数据库的当前持续性上下文中后或在向其应用了刷新操作后调用重写方法onPostLoad。

`PostInsertEventListener`	

监听save之后的事件，要使用该功能只需实现该接口并重写onPostInsert方法，并在该方法里面写保存之后需要做的事就行

`PreInsertEventListener`

监听save之前的事件，要使用该功能只需实现该接口并重写onPreInsert方法，并在该方法里面写保存之前需要做的事就行

`PostUpdateEventListener`	

监听update之后的事件，要使用该功能只需实现该接口并重写     onPostUpdate方法，并在该方法里面写更新之后需要做的事就行

`PreUpdateEventListener`	

监听update之前的事件，要使用该功能只需实现该接口并重写onPreUpdate方法，并在该方法里面写更新之前需要做的事就行

`PostDeleteEventListener`	

监听remove之后的事件，要使用该功能只需实现该接口并重写onPostRemove方法，并在该方法里面写移除之后需要做的事就行

`PreDeleteEventListener`

监听remove之前的事件，要使用该功能只需实现该接口并重写onPreRemove方法，并在该方法里面写移除之前需要做的事就行

# 项目应用

引入包，由于是spring data jpa提供的，故只需引入jpa的包即可

```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-jpa</artifactId>
</dependency>
```

定义监听类

定义监听类AuditLogListener,这里根据具体要监听什么事件，去实现对应的接口，这里实现PostInsertEventListener, PostUpdateEventListener, PostDeleteEventListener这三个接口并重写对应的方法来实现对新增、修改、删除进行监听：

```java
import org.hibernate.event.spi.*;
import org.hibernate.persister.entity.EntityPersister;
import org.hibernate.persister.entity.SingleTableEntityPersister;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * 审计事件监听
 */
@Component
public class AuditLogListener implements PostInsertEventListener, PostUpdateEventListener, PostDeleteEventListener {

    /**
     * 新增
     * @param postInsertEvent
     */
    @Override
    public void onPostInsert(PostInsertEvent postInsertEvent) {
      //数据库
      String dbName = postInsertEvent.getSession().getJdbcServices().getJdbcEnvironment()
        .getCurrentCatalog().getText();
      //表
      String tableName = ((SingleTableEntityPersister)postInsertEvent.getPersister()).getTableName();
      //id
      String id = postInsertEvent.getId();
      //字段
      String[] propertyNames = postInsertEvent.getPersister().getPropertyNames();
      //数据
      Object[] state = postInsertEvent.getState();     
    }

    /**
     * 更新
     * @param postUpdateEvent
     */
    @Override
    public void onPostUpdate(PostUpdateEvent postUpdateEvent) {
      String dbName = postUpdateEvent.getSession().getJdbcServices().getJdbcEnvironment()
        .getCurrentCatalog().getText();
      String tableName = ((SingleTableEntityPersister)postUpdateEvent.getPersister()).getTableName();
      //字段
      String[] propertyNames = postUpdateEvent.getPersister().getPropertyNames();
      //修改后的数据
      Object[] currentState = postUpdateEvent.getState();
      //修改前的数据
      Object[] previousState = postUpdateEvent.getOldState();
      //改变的数组
      int[] dirtyProperties = postUpdateEvent.getDirtyProperties(); 
    }

    /**
     * 删除
     * @param postDeleteEvent
     */
    @Override
    public void onPostDelete(PostDeleteEvent postDeleteEvent) {
      String dbName = postDeleteEvent.getSession().getJdbcServices().getJdbcEnvironment()
        .getCurrentCatalog().getText();
      String tableName = ((SingleTableEntityPersister)postDeleteEvent.getPersister()).getTableName();
      String[] propertyNames = postDeleteEvent.getPersister().getPropertyNames();
      Object[] state = postDeleteEvent.getDeletedState();     
    }

    @Override
    public boolean requiresPostCommitHanding(EntityPersister entityPersister) {
        return false;
    }

}
```

# 将监听器注册进对应的数据源

```java
import org.hibernate.event.service.spi.EventListenerRegistry;
import org.hibernate.event.spi.EventType;
import org.hibernate.internal.SessionFactoryImpl;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import javax.persistence.EntityManagerFactory;

/**
 * 配置监听
 */
@Component
public class ListenerConfig {

    @Autowired
    @Qualifier("basicEntityManagerFactory")
    private EntityManagerFactory emf;

    @Autowired
    private AuditLogListener auditLogListener;


    @PostConstruct
    protected void init() {
        SessionFactoryImpl sessionFactory = emf.unwrap(SessionFactoryImpl.class);
        EventListenerRegistry registry = sessionFactory.getServiceRegistry().
          getService(EventListenerRegistry.class);
        registry.getEventListenerGroup(EventType.POST_INSERT).appendListener(auditLogListener);
        registry.getEventListenerGroup(EventType.POST_UPDATE).appendListener(auditLogListener);
        registry.getEventListenerGroup(EventType.POST_DELETE).appendListener(auditLogListener);
    }
}
```

# 在需要监听的entity上加注解@EntityListeners

```java
@Getter
@Setter
@Entity
@Table(name="product")
@EntityListeners(AuditLogListener.class)
public class Product {

    @Column(name = "`coid`")
    private Integer coid;

    @Column(name = "`ncoid`")
    private Integer ncoid;

}
```

# 启动类加上@EnableJpaAuditing注解

```java
@SpringBootApplication
@EnableJpaAuditing
public class ConsoleBasicApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConsoleBasicApplication.class, args);
        System.out.println("started");
    }
}
```

# 项目参考

https://github.com/FintechLabs/spring-boot-audit-log/blob/master/src/main/java/com/fintechlabs/auditlogs/audit/AuditLogListener.java

