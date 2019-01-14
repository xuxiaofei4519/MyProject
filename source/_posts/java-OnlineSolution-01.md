---
title: 记一次线上问题解决
date: 2019-01-14 20:33:02
copyright: true
tags:
 - java
categories:
 - java
---

{% cq %} 
4号早晨，邮箱大量探活失败报警和cpu占用过高报警，因此将此次线上问题解决记录下来。
{% endcq %}
<!-- more -->

#### 问题描述
> 2019-01-04企业微信邮箱收到大量运维报警邮件，cpu占用过高和服务探活失败报警，通过登录线上服务器发现，该服务器为收银台线上task机器。


#### 问题解决过程

1. 登录该服务器，使用top查看cpu占用过高的进程。
2. 使用top -Hp pid 查看该进程中各个线程占用资源情况，发现有两三个线程占用cpu时间超过300多个小时，并且cpu占用率超过百分之百
3. 因此将占用过高的线程的pid转换成16进制   printf %x <pid>
4. jstack pid | grep 0x16进制
   发现：
{% qnimg /java-OnlineSolution-01/java-OnlineSolution01.jpg %}
   该占用cpu过高的线程为GC线程，因此查看gc情况
{% qnimg /java-OnlineSolution-01/java-OnlineSolution02.jpg %}
5. jstat -gcutil pid 1000 100
{% qnimg /java-OnlineSolution-01/java-OnlineSolution03.jpg %}
   上图不是问题产生时的截图，只是用于展示命令
   线上解决的时候看到E(eden)和O(老年代)占用内存过大，说明代码中存在大量对象没有被销毁的逻辑

6. 使用jmap -heap pid 查看进程堆内存使用情况，包括使用的GC算法、堆配置参数和各代中堆内存使用情况。
   使用jmap -histo:live pid  查看堆内存中的对象数目、大小统计直方图，如果带上live则只统计活对象。
   {% qnimg /java-OnlineSolution-01/java-OnlineSolution04.jpg %}
   发现大量LinkedList和Span实例，回顾系统最新一次上线业务，该业务接入全链路，使用到了内部的silt4j包，因此查看代码，发现存在没有手动关闭全链路追踪的情况，导致实例一直占用堆内存。
   {% qnimg /java-OnlineSolution-01/java-OnlineSolution05.jpg %}
   上面代码中如果执行到continue，上面MDCUtil.strikeFullLink()就不会被关闭，久而久之堆中内存就会占用过高，导致频繁GC，进而导致cpu飙升
   