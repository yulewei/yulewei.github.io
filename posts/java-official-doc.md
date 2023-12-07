---
title: Java 官方文档整理 
date: 2016-11-20 03:21:32
categories: Java
tags: [Java, JVM]
---

本文整理一些学习Java需要翻阅或必须的官方文档资料。

<!--more-->

# Java SE文档

Java SE Technologies at a Glance <http://www.oracle.com/technetwork/java/javase/tech/index.html>

<http://docs.oracle.com/javase/8/>
<http://docs.oracle.com/javase/8/docs/index.html> Platform Overview平台概览图
<http://docs.oracle.com/javase/8/javase-books.htm>

Java Platform Overview (docs/technotes/[guides](http://docs.oracle.com/javase/8/docs/technotes/guides/))
1. Java Programming Language (guides/[language](http://docs.oracle.com/javase/8/docs/technotes/guides/language/index.html))
2. Java Virtual Machine Technology (guides/[vm](http://docs.oracle.com/javase/8/docs/technotes/guides/vm/index.html))
3. JDK Tools and Utilities (technotes/[tools](http://docs.oracle.com/javase/8/docs/technotes/tools/index.html))

Java SE API：

1. Base Libraries (guides/[\#base](http://docs.oracle.com/javase/8/docs/technotes/guides/#base)):
  - Lang and Util Packages: [lang and util](http://docs.oracle.com/javase/8/docs/technotes/guides/lang/index.html) (java.lang & java.util), [Math](http://docs.oracle.com/javase/8/docs/technotes/guides/math/index.html) (java.lang.Math), [Collections](http://docs.oracle.com/javase/8/docs/technotes/guides/collections/index.html) (java.util), Reference Objects, Regular Expressions (java.util.regex), [Logging](http://docs.oracle.com/javase/8/docs/technotes/guides/logging/index.html) (java.util.logging), [Management](http://docs.oracle.com/javase/8/docs/technotes/guides/management/index.html) (java.lang.management), [Instrumentation](http://docs.oracle.com/javase/8/docs/technotes/guides/instrumentation/index.html) (java.lang.instrument), [Concurrency Utilities](http://docs.oracle.com/javase/8/docs/technotes/guides/concurrency/index.html) (java.lang.Thread, java.util.concurrent), [Reflection](http://docs.oracle.com/javase/8/docs/technotes/guides/reflection/index.html) (java.lang.reflect), [Package Versioning](http://docs.oracle.com/javase/8/docs/technotes/guides/versioning/index.html), [Preferences API](http://docs.oracle.com/javase/8/docs/technotes/guides/preferences/index.html) (java.util.prefs), [JAR](http://docs.oracle.com/javase/8/docs/technotes/guides/jar/index.html) (java.util.jar), Zip (java.util.zip)
  - Other Base Packages: [Beans](http://docs.oracle.com/javase/8/docs/technotes/guides/beans/index.html) (java.beans), [Security](http://docs.oracle.com/javase/8/docs/technotes/guides/security/index.html) (java.security & javax.crypto), [Serialization](http://docs.oracle.com/javase/8/docs/technotes/guides/serialization/index.html) (java.io), [Extension Mechanism](http://docs.oracle.com/javase/8/docs/technotes/guides/extensions/index.html), [JMX](http://docs.oracle.com/javase/8/docs/technotes/guides/jmx/index.html), [XML JAXP](http://docs.oracle.com/javase/8/docs/technotes/guides/xml/index.html), [Networking](http://docs.oracle.com/javase/8/docs/technotes/guides/net/index.html) (java.net & javax.net), [Override Mechanism](http://docs.oracle.com/javase/8/docs/technotes/guides/standards/index.html), [JNI](http://docs.oracle.com/javase/8/docs/technotes/guides/jni/index.html), [Date and Time](http://docs.oracle.com/javase/8/docs/technotes/guides/datetime/index.html) (java.time), [Input/Output](http://docs.oracle.com/javase/8/docs/technotes/guides/io/index.html) (java.io & java.nio), [Internationalization](http://docs.oracle.com/javase/8/docs/technotes/guides/intl/index.html)
2. Integration Libraries (guides/[\#integration](http://docs.oracle.com/javase/8/docs/technotes/guides/#integration)): [IDL](http://docs.oracle.com/javase/8/docs/technotes/guides/idl/index.html) (CORBA, org.omg.\*), [JDBC](http://docs.oracle.com/javase/8/docs/technotes/guides/jdbc/index.html) (java.sql & javax.sql), [RMI](http://docs.oracle.com/javase/8/docs/technotes/guides/rmi/index.html) (java.rmi), [RMI-IIOP](http://docs.oracle.com/javase/8/docs/technotes/guides/rmi-iiop/index.html) (org.omg.\*) [JNDI](http://docs.oracle.com/javase/8/docs/technotes/guides/jndi/index.html) (javax.naming), [Scripting](http://docs.oracle.com/javase/8/docs/technotes/guides/scripting/index.html) (javax.script)
3. User Interface Libraries (guides/[\#userinterface](http://docs.oracle.com/javase/8/docs/technotes/guides/#userinterface)): Swing, Java 2D, AWT, Accessibility, Drag and Drop, Input Methods, Image I/O, Print Service, Sound

<http://docs.oracle.com/javase/8/docs/technotes/guides/scripting/>

**中文API文档：**
<http://download.oracle.com/technetwork/java/javase/6/docs/zh/api/>
<http://www.cjsdn.net/Doc/JDK60/>

---

# Java虚拟机

<https://en.wikipedia.org/wiki/Template:Java_Virtual_Machine>
<https://en.wikipedia.org/wiki/Java_performance>

**JDK源码**：
<https://github.com/dmlloyd/openjdk>
<https://github.com/openjdk-mirror/jdk7u-jdk>
<https://github.com/openjdk-mirror/jdk7u-hotspot>

**虚拟机与性能**
[http://en.wikipedia.org/wiki/Template:Java\_%28software\_platform%29](http://en.wikipedia.org/wiki/Template:Java_%28software_platform%29)
[http://en.wikipedia.org/wiki/Template:Java\_Virtual\_Machine](http://en.wikipedia.org/wiki/Template:Java_Virtual_Machine)
[http://en.wikipedia.org/wiki/Java\_performance](http://en.wikipedia.org/wiki/Java_performance)
<http://openjdk.java.net/groups/hotspot/>

**Java SE HotSpot at a Glance**，[link](http://www.oracle.com/technetwork/java/javase/tech/index-jsp-136373.html)
1. HotSpot Engine Architecture，[link](http://www.oracle.com/technetwork/java/whitepaper-135217.html)
2. HotSpot Thread Implementation (Solaris)，[link](http://www.oracle.com/technetwork/java/threads-140302.html)
3. HotSpot Garbage Collection，[link](http://www.oracle.com/technetwork/java/javase/tech/index-jsp-140228.html)
  - **Memory Management Whitepaper[pdf]**，[link](http://www.oracle.com/technetwork/java/javase/tech/memorymanagement-whitepaper-1-150020.pdf)：最权威、最完整文档
  - Garbage Collector Ergonomics，[link](http://docs.oracle.com/javase/1.5.0/docs/guide/vm/gc-ergonomics.html)
  - Garbage Collection Tuning，[link](http://www.oracle.com/technetwork/java/javase/gc-tuning-6-140523.html)
  - Garbage First ("G1") Garbage Collector，[link](http://www.oracle.com/technetwork/java/javase/tech/g1-intro-jsp-135488.html)
4. HotSpot Ergonomics，[link](http://www.oracle.com/technetwork/java/ergo5-140223.html)
5. HotSpot Performance and Tuning，[link](http://www.oracle.com/technetwork/java/performance-138178.html)
  - 2007.10, Java SE 6.0 Performance White Paper，[link](http://www.oracle.com/technetwork/java/6-performance-137236.html)
  - 2005.03, J2SE 5.0 Performance White Paper，[link](http://www.oracle.com/technetwork/java/5-136747.html)
8. HotSpot Publications，[link](http://www.oracle.com/technetwork/java/javase/tech/publications-140132.html)

**Troubleshooting Java SE 8，[link](http://www.oracle.com/technetwork/java/javase/index-138283.html) 资料汇总**

1. Java Troubleshooting Guide Java SE 8，[link](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/toc.html)
2. Troubleshooting Guide for Java SE 6 with HotSpot VM，[link](http://www.oracle.com/technetwork/java/javase/memleaks-137499.html)
3. Troubleshooting Guide for HotSpot VM (JDK 7)，[link](http://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/%0A)


