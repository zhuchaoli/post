---
title: k8s环境下如何本地调试
categories: Java
date: 2021-08-07 21:44:04
tags: k8s
---

# 背景

Kubernetes 应用程序通常由多个独立的服务组成，每个服务都在自己的容器中运行。 在远端的 Kubernetes 集群上开发和调试这些服务可能很麻烦。

这个时候我们可以考虑关闭k8s，通过手动设置服务列表的方式来调用相关的服务 <!-- more -->

# 配置

在服务调用方，关闭服务发现，同时指定服务提供方的地址

例如：A调用B，在A的配置中加入下面的配置，同时指定B的地址

```properties
spring.cloud.kubernetes.enabled=false
spring.cloud.kubernetes.discovery.enabled=false
spring.cloud.kubernetes.ribbon.enabled=false
service.ribbon.listOfServers=localhost:9000
```

其中service.ribbon.listOfServers 为配置的服务列表，可配置多条逗号隔开

将前缀service替换为服务提供方的名称，更改为自己项目ip+端口即可

然后启动A、B服务，就可以直接实现服务间的调用，方便本地的开发调试

# 其他方案

KtConnect提供了本地和测试环境集群的双向互联能力。在这篇文档里，我们将使用一个简单的示例，来快速演示通过KtConnect完成本地直接访问集群中的服务、以及将集群中指定服务的请求转发到本地的过程。

https://alibaba.github.io/kt-connect/#/zh-cn/quickstart

#  参考文档

https://projects.spring.io/spring-cloud/spring-cloud.html#spring-cloud-ribbon-without-eureka

https://cloud.spring.io/spring-cloud-static/spring-cloud-kubernetes/1.1.0.M2/reference/html/

