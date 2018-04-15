---
title: EnumMap详解
copyright: true
tags:
 - java
categories:
 - java
---

{% cq %} 
前段时间学习spring、tomcat的源码非常吃力，发现很多东西都是因为自己的Java基础不够牢固,
因此搭建了此博客开始重点关注java基础的相关知识。这篇文章主要分析自己很少使用的EnumMap。
{% endcq %}

<!-- more -->

### EnumMap的继承类与实现接口

![截图](/image/java-EnumMap/java-EnumMap01.png)


### EnumMap内部的方法
![截图](/image/java-EnumMap/java-EnumMap02.png)


### 类定义
{% codeblock lang:java %}
public class EnumMap<K extends Enum<K>, V> extends AbstractMap<K, V> implements java.io.Serializable, Cloneable{
    private final Class<K> keyType;
    private transient K[] keyUniverse;
    private transient Object[] vals;
    private transient int size = 0;
}
{% endcodeblock %}


  


