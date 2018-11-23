---
title: this引用溢出导致的安全问题
date: 2018-11-08 14:20:09
copyright: true
tags:
 - java
categories:
 - java
---

{% cq %} 
最近在读并发相关的文章，了解到一个重要知识点，拿出来分享给大家
{% endcq %}
<!-- more -->

* * *

#### this引用溢出示例
ThisEscape类
```java
public class ThisEscape {  
    private String name = null;  
  
    public ThisEscape(EventSource source) {  
        source.registerListener(new EventListener() {  
  
            public void onEvent(Event event) {  
                doSomething(event);  
            }  
  
        });
        //可以看出在上面的registerListener方法中调用了listener.onEvent（）方法，
        //onEvent方法里面调用了dosomething但这个时候name还没有被初始化，
        //因此执行name.toString的时候肯定会报空指针，其实就是this引用溢出了
        name = "TEST";  
    }  
  
    /** 
     * 
     * @param event 
     */  
    protected void doSomething(Event event) {  
        System.out.println(name.toString());  
    }  
}  
```
##### EventListener接口
```java
import java.awt.Event; 

public interface EventListener {  
    public void onEvent(Event event);  
  
} 
```
##### EventSource接口
```java
public class EventSource {  
  
    public void registerListener(EventListener listener) {  
        listener.onEvent(null);  
    }  
  
}  
```
##### main方法
```java
public class Client {  
  
    /** 
     * 
     * @param args 
     * @throws InterruptedException 
     */  
    public static void main(String[] args) throws InterruptedException {  
        EventSource es = new EventSource();  
        new ThisEscape(es);  
    }  
  
}  
```
##### 运行结果:
{% qnimg /java-ThisEscape/java-ThisEscape-01.png %}
> 我们可以看出再name初始化之前，我们就使用了ThisEscape实例即this，由于new EventListener(){}是匿名内部类，在创建匿名内部类的时候其实是生成外部类class文件和内部类class文件，内部类class文件中有外部类的引用即this引用，因此还没初始化完成就调用了this引用，造成引用溢出。

#### 正确的编程
> 解决方法：在初始化完成后再使用该类实例的方法或变量。
##### SafeListener类
```java
public class SafeListener {  
    //这里使用final来保证成员变量的值不变，保证匿名内部类和外部环境局部变量保持同步,
    //也就是不允许对EventListener修改,保证构造过程中的安全问题。
    private final EventListener listener;  
    private String              name = null;  
  
    private SafeListener() {  
        listener = new EventListener() {  
  
            public void onEvent(Event event) {  
                doSomething();  
            }  
        };  
        name = "TEST";  
    }  
  
    public static SafeListener newInstance(EventSource eventSource) {  
        SafeListener safeListener = new SafeListener();  
        eventSource.registerListener(safeListener.listener);  
        return safeListener;  
    }  
  
    protected void doSomething() {  
        System.out.println(name.toString());  
    }  
}  
```
##### client方法
```java
public class Client {

    public static void main(String[] args) throws InterruptedException {
        EventSource es = new EventSource();
//        new ThisEscape(es);
        SafeListener.newInstance(es);
    }
}

```
##### 执行结果:
{% qnimg /java-ThisEscape/java-ThisEscape-02.png %}

> 这样在正式初始化完成后，再调用不会出现引用溢出的安全性问题。