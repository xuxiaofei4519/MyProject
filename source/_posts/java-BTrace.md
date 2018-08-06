---
title: BTrace实战
date: 2018-08-05 20:40:09
copyright: true
tags:
 - java
categories:
 - java
---

{% cq %} 
最近了解到关于java动态追踪工具BTrace的相关内容，决定实战一下，本文是针对BTrace的简单实战，未涉及过多详细内容
{% endcq %}
<!-- more -->

### BTrace简介
>Btrace (Byte Trace)是sun推出的一款java 动态、安全追踪（监控）工具，可以不停机的情况下监控线上情况，并且做到最少的侵入，占用最少的系统资源。BTrace应用较为广泛的原因应该是其安全性和无侵入性，其中热交互技术，使得我们无需启动Agent的情况下动态跟踪分析，其安全性不会导致对目标Java进程的任何破坏性影响，使得BTrace成为我们线上产品问题定位的利器。无侵入性无需我们对原有代码做任何修改，降低上线风险和测试成本，并且无需重启启动目标Java进程进行Agent加载即可动态分析和跟踪目标程序，可以说BTrace可以满足大部分的应用场景。

### BTrace安装
- 首先下载[release](https://github.com/btraceio/btrace)版本
- 配置环境变量
 ![截图](/image/java-BTrace/java-BTrace01.png)
- 验证是否安装成功
 ![截图](/image/java-BTrace/java-BTrace02.png)
 
### BTrace实战
- 测试用例(间隔一秒打印当前时间戳)
 ![截图](/image/java-BTrace/java-BTrace03.png)
- 追踪脚本
 ![截图](/image/java-BTrace/java-BTrace04.png)
- 追踪结果
 ![截图](/image/java-BTrace/java-BTrace05.png)
 
### BTrace使用限制
- 不能创建对象
- 不能创建数组
- 不能抛出和捕获异常
- 不能调用任何对象方法和静态方法
- 不能给目标程序中的类静态属性和对象的属性进行赋值
- 不能有外部、内部和嵌套类
- 不能有同步块和同步方法
- 不能有循环(for, while, do..while)
- 不能继承任何的类
- 不能实现接口
- 不能包含assert断言语句



