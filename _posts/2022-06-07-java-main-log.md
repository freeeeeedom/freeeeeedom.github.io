---
layout: post
title: JAVA使用main方法测试执行HttpClient日志关闭Debug
date: 2022-06-07
categories: blog
tags: []
description: Java是最好的语言。
---
## 简述
在写函数、spring项目等测试第三方接口使用main方法测试接口可用性时，会发现有大量HttpClient包中自带的debug级别日志，这些都不是我们需要的日志，因此我们需要想办法修改日志级别！
![](http://freeeeeedom.github.io/img/html-log.png)
![](http://freeeeeedom.github.io/img/html-log-2.png)

## 处理方式
HttpClient的日志模组是logback，在类加载时增加以下代码，将所有logback的日志级别改为ERROR
```java
((LoggerContext)LoggerFactory.getILoggerFactory())
                .getLoggerList()
                .forEach(logger -> logger.setLevel(Level.ERROR));
```
当然也可以指定HttpClient的包来做到精确控制
```java
((LoggerContext)LoggerFactory.getILoggerFactory())
                .getLogger("org.apache.http")
                .setLevel(Level.ERROR);
```
以上涉及logback日志级别调整只需要引入两个类即可
```java
import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.LoggerContext;
```
