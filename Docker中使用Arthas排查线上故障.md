---
title: Docker中使用Arthas排查线上故障
date: 2021-08-07 16:34:00
tags: Arthas
categories: Java
---

# Arthas能为你做什么？

`Arthas` 是Alibaba开源的Java诊断工具，深受开发者喜爱。<!-- more -->

当你遇到以下类似问题而束手无策时，`Arthas`可以帮助你解决：

1. 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
2. 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
3. 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
4. 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
5. 是否有一个全局视角来查看系统的运行状况？
6. 有什么办法可以监控到JVM的实时运行状态？
7. 怎么快速定位应用的热点，生成火焰图？
8. 怎样直接从JVM内查找某个类的实例？

`Arthas`支持JDK 6+，支持Linux/Mac/Windows，采用命令行交互模式，同时提供丰富的 `Tab` 自动补全功能，进一步方便进行问题的定位和诊断。

# 在Docker中使用Arthas

很多时候，应用在docker里出现arthas无法工作的问题，是因为应用没有安装 JDK ，而是安装了 JRE 。如果只安装了 JRE，则会缺少很多JAVA的命令行工具和类库，Arthas也没办法正常工作。所以需要更换JDK。

```dockerfile
FROM openjdk:8-jdk
```

或者：

```dockerfile
FROM openjdk:8-jdk-alpine
```

## 提供三种使用Arthas的方式

### 诊断Docker里的Java进程

```shell
docker exec -it  ${containerId} /bin/bash -c "wget https://arthas.aliyun.com/arthas-boot.jar && java -jar arthas-boot.jar"
```

### 诊断k8s里容器里的Java进程

```shell
kubectl exec -it ${pod} --container ${containerId} -- /bin/bash -c "wget https://arthas.aliyun.com/arthas-boot.jar && java -jar arthas-boot.jar"
```

### 把Arthas安装到基础镜像里

```dockerfile
FROM openjdk:8-jdk-alpine

# copy arthas
COPY --from=hengyunabc/arthas:latest /opt/arthas /opt/arthas
```

增加 arthas 后在 docker 环境中需要使用 tini 作为父进程来收割僵尸进程

![](https://p.pstatp.com/origin/pgc-image/91a4596505224aeaa5836f18d180f754)

# 开始诊断

## trace

> 方法内部调用路径，并输出方法路径上的每个节点上耗时
>
> https://arthas.aliyun.com/doc/trace.html

![1](https://p.pstatp.com/origin/pgc-image/643ca064e703417f92b4650c2d430d5d)

官方文档： https://arthas.aliyun.com/doc/

