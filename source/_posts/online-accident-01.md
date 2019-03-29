---
title: 记线上事故：dubbo Filter调用异常
date: 2019-03-29 17:20:06
copyright: true
tags:
 - 线上事故
categories:
 - 线上事故
---

{% cq %} 
今天遇到的线上事故，因此记录下来，防止自己也会犯类似错误
{% endcq %}
<!-- more -->


#### 问题描述
2019-03-28晚7点，业务需求正常上线，QA灰度回归验证时，老业务接口出现dubbo自定义Filter异常，异常信息：

```java
com.alibaba.dubbo.registry.integration.RegistryDirectory$InvokerDelegate@3b232c1f] for service com.jiupai.paybase.api.interfaces.charge.IQuickChargeService method sendSms on consumer 999.999.999.99 use dubbo version 2.6.5, but no luck to perform the invocation. Last error is: class com.jiupai.paybase.api.dto.response.charge.QuickPaySmsResp declares multiple JSON fields named amount
 at com.alibaba.dubbo.rpc.cluster.support.FailfastClusterInvoker.doInvoke(FailfastClusterInvoker.java:53) ~[dubbo-2.6.5.jar:2.6.5]
 at com.alibaba.dubbo.rpc.cluster.support.AbstractClusterInvoker.invoke(AbstractClusterInvoker.java:244) ~[dubbo-2.6.5.jar:2.6.5]
	at com.alibaba.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker.invoke(MockClusterInvoker.java:75) ~[dubbo-2.6.5.jar:2.6.5]
	at com.alibaba.dubbo.rpc.proxy.InvokerInvocationHandler.invoke(InvokerInvocationHandler.java:52) ~[dubbo-2.6.5.jar:2.6.5]
	at com.alibaba.dubbo.common.bytecode.proxy18.sendSms(proxy18.java) ~[dubbo-2.6.5.jar:2.6.5]
	at com.jiupai.cashier.biz.service.sendmessage.impl.MessageServiceImpl.sendAuthCodeMessage(MessageServiceImpl.java:110) ~[cashier-biz-1.3.1.jar:na]
	at com.jiupai.cashier.web.controller.chongtizhaun.SendMessageController.sendAuthCodeMessage(SendMessageController.java:56) [SendMessageController.class:na]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.7.0_79]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57) ~[na:1.7.0_79]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.7.0_79]
	at java.lang.reflect.Method.invoke(Method.java:606) ~[na:1.7.0_79]
	at org.springframework.web.method.support.InvocableHandlerMethod.invoke(InvocableHandlerMethod.java:215) [spring-web-3.2.9.RELEASE.jar:3.2.9.RELEASE]
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:132) [spring-web-3.2.9.RELEASE.jar:3.2.9.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:104) [spring-webmvc-3.2.9.RELEASE.jar:3.2.9.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandleMethod(RequestMappingHandlerAdapter.java:745) [spring-webmvc-3.2.9.RELEASE.jar:3.2.9.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:685) [spring-webmvc-3.2.9.RELEASE.jar:3.2.9.RELEASE]
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:80) [spring-webmvc-3.2.9.RELEASE.jar:3.2.9.RELEASE]
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:919) [spring-webmvc-3.2.9.RELEASE.jar:3.2.9.RELEASE]
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:851) [spring-webmvc-3.2.9.RELEASE.jar:3.2.9.RELEASE]
 at java.lang.Thread.run(Thread.java:745) [na:1.7.0_79]
Caused by: java.lang.IllegalArgumentException: class com.jiupai.paybase.api.dto.response.charge.QuickPaySmsResp declares multiple JSON fields named amount
	at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory.getBoundFields(ReflectiveTypeAdapterFactory.java:166) ~[gson-2.6.1.jar:na]
	at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory.create(ReflectiveTypeAdapterFactory.java:96) ~[gson-2.6.1.jar:na]
	at com.google.gson.Gson.getAdapter(Gson.java:416) ~[gson-2.6.1.jar:na]
	at com.google.gson.Gson.toJson(Gson.java:653) ~[gson-2.6.1.jar:na]
	at com.google.gson.Gson.toJson(Gson.java:640) ~[gson-2.6.1.jar:na]
	at com.google.gson.Gson.toJson(Gson.java:595) ~[gson-2.6.1.jar:na]
	at com.google.gson.Gson.toJson(Gson.java:575) ~[gson-2.6.1.jar:na]
	at com.jiupai.cornerstone.util.json.JsonUtils.objectToJsonStr(JsonUtils.java:49) ~[cornerstone-util-1.0.3.jar:na]
	at com.jiupai.cornerstone.rpc.filter.ConsumerFilter.invoke(ConsumerFilter.java:52) ~[cornerstone-rpc-allinone-1.1.7.2.jar:na]
	at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:72) ~[dubbo-2.6.5.jar:2.6.5]
	at com.jiupai.cornerstone.rpc.filter.TraceInfoFilter.invoke(TraceInfoFilter.java:63) ~[cornerstone-rpc-allinone-1.1.7.2.jar:na]
	at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:72) ~[dubbo-2.6.5.jar:2.6.5]
	at com.alibaba.dubbo.rpc.filter.ConsumerContextFilter.invoke(ConsumerContextFilter.java:49) ~[dubbo-2.6.5.jar:2.6.5]
	at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:72) ~[dubbo-2.6.5.jar:2.6.5]
	at com.alibaba.dubbo.rpc.listener.ListenerInvokerWrapper.invoke(ListenerInvokerWrapper.java:77) ~[dubbo-2.6.5.jar:2.6.5]
	at com.alibaba.dubbo.rpc.protocol.InvokerWrapper.invoke(InvokerWrapper.java:56) ~[dubbo-2.6.5.jar:2.6.5]
	at com.alibaba.dubbo.rpc.cluster.support.FailfastClusterInvoker.doInvoke(FailfastClusterInvoker.java:48) ~[dubbo-2.6.5.jar:2.6.5]
	... 54 common frames omitted
```


为了安全起见:将上面日志IP变为999.999.999.99

初步确认是由于下游API包重复字段导致Gson工具Object转Json时出现异常，但是和下游RD确认，接口API未改动，且都是老业务接口，未做改动，为避免阻塞上线，下游RD去掉重复字段，重新修改版本，灰度回滚，升级版本号，重新上线，灰度回归验证通过。最终成功上线。

2019-03-29中午，线上购买理财业务出现异常，查询日志如下:
{% qnimg /online-accident-01/online-accident-01-1.png %}

还是同样错误，并且此业务和昨天上线需求没有任何关系，观察日志，发现dubbo调用时进入了cornerstone项目的filter，查看cornerstone项目下该包的Filter，发现都是@Activate，且cornerstone/cornerstone-rpc/cornerstone-rpc-filter/src/main/resources/META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Filter中也有指定，此时我非常困惑于为什么会走这么多的filter，因为cornerstone是整个支付的基础项目，有可能是其他业务必须的。询问下游RD，下游也不清楚这些filter是干什么用的。回想这个异常是出自老业务，昨天上线的需求和该业务没有一点关系。因此最终确定，就是包引用问题。查看上线前下游API版本中的pom和上线后下游API版本中的pom，发现多出如下内容:
{% qnimg /online-accident-01/online-accident-01-2.png %}
该包正好是日志中堆栈信息所在的包，最终和下游确认，该包不能对外暴露。但是由于下游API包需要该依赖，因此添加:
{% qnimg /online-accident-01/online-accident-01-3.png %}
这样防止上游依赖，最终修改后，版本升级，bug上线。

#### 反思
1. 分析问题时要果断，要带着问题去一步一步思考是什么，为什么。
2. 下游的此次修改也给我一个警示，对外不能随便暴露API包，尤其网关类系统在使用的时候，要格外注意依赖对自身系统的侵入性，否则一不小心就会出现服务大面积崩溃。
3. API内部使用的包不想被外部使用可以在dependency添加scope域，防止外部依赖。