---
title: ssm开发遇到的坑小记
date: 2017-12-24 17:16:59
tags: 
- spring
categories: 
- java
---

## spring dateSources bean创建错误

在我开启WEB服务器的时候，控制台报出了`ClassNotFoundException: org.aspect.weaver....`,等一系列关于spring dateSources bean创建错误。
```
caused by: java.lang.NoClassDefFoundError: org/aspectj/weaver/reflect/ReflectionWorld$ReflectionWorldException
	at java.lang.Class.getDeclaredConstructors0(Native Method)
	at java.lang.Class.privateGetDeclaredConstructors(Unknown Source)
	at java.lang.Class.getConstructor0(Unknown Source)
	at java.lang.Class.getDeclaredConstructor(Unknown Source)
	at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:78)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateBean(AbstractAutowireCapableBeanFactory.java:1030)
	... 47 more
Caused by: java.lang.ClassNotFoundException: org.aspectj.weaver.reflect.ReflectionWorld$ReflectionWorldException
	at org.apache.catalina.loader.WebappClassLoader.loadClass(WebappClassLoader.java:1680)
	at org.apache.catalina.loader.WebappClassLoader.loadClass(WebappClassLoader.java:1526)
	... 53 more
```

### 解决过程

仔细看报错内容可以发现是由于缺少`spring-aspects`库导致，于是在`maven`依赖中添加`spring-aspects`相关的依赖包.
```
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>${spring.version}</version>
</dependency>
```
然后重启服务器，问题解决。

## <fmt:formatDate> problem

错误提示
```
Unable to convert string "${item.startTime}" to class "java.util.Date" for attribute "value": 
Property Editor not registered with the PropertyEditorManager
```

页面jsp代码
`<td><fmt:formatDate value="${item.createtime}" pattern="yyyy-MM-dd HH:mm:ss"/></td>`

### 解决过程

这个问题出现的时候我以为是标签哪里有问题，后来在发现有由于`web.xm`声明的问题：
```
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >
```
这样写会有问题，改为
```
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://java.sun.com/xml/ns/javaee"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         metadata-complete="true" version="3.0">

</web-app>
```
便不会出现这个问题了。因为Tomcat的支持web.xml的Servlet是2.5版本。