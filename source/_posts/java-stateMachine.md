---
title: EnumMap状态机实现
date: 2018-04-22 23:49:25
copyright: true
tags:
 - java
categories:
 - java
---
{% cq %}
根据公司老大提示，查阅资料发现状态机其实是一种设计模式，能够很大程度上减少我们if else的条件判断，尤其是在做状态转换的时候，本篇文章通过EnumMap做一个状态机的简单实现。
{% endcq %}

<!-- more -->

> 根据公司项目(P2P网关)中的状态，简单实现一个状态机

### 状态定义枚举类
{% codeblock lang:java %}
/**
 * 状态定义
 */
public enum StatusDefine {
    SUCCESS,
    FAILED,
    DEALING,
    UNKNOW;
}
{% endcodeblock %}

### 具体操作接口
{% codeblock lang:java %}
/**
 * 具体操作接口
 */
public interface Action {
    /**
     * 转换状态方法
     */
    public void convertStatus();
}
{% endcodeblock %}

### 状态转换枚举类
{% codeblock lang:java %}
public enum StatusConvert{
    /** 失败转换成成功 */
    FAIL_TO_SUCCESS(StatusDefine.FAILED,StatusDefine.SUCCESS){
        @Override
        public void todoConvertStatus() {
            doSomething(new Action() {
                @Override
                public void convertStatus() {
                    System.out.println("失败->成功");
                }
            });
        }
    },
    DEALING_TO_SUCCESS(StatusDefine.DEALING,StatusDefine.SUCCESS){
        @Override
        public void todoConvertStatus() {
            doSomething(new Action() {
                @Override
                public void convertStatus() {
                    System.out.println("处理中->成功");
                }
            });
        }
    },
    UNKNOWN_TO_SUCCESS(StatusDefine.UNKNOW,StatusDefine.SUCCESS){
        @Override
        public void todoConvertStatus() {
            doSomething(new Action() {
                @Override
                public void convertStatus() {
                    System.out.println("未知->成功");
                }
            });
        }
    };
    public static final EnumMap<StatusDefine, Map<StatusDefine, StatusConvert>> allStatus = new EnumMap<StatusDefine, Map<StatusDefine, StatusConvert>>(StatusDefine.class);
    private StatusDefine start;
    private StatusDefine target;
    static {
        for (StatusDefine statusDefine : StatusDefine.values()){
            allStatus.put(statusDefine,new EnumMap<StatusDefine, StatusConvert>(StatusDefine.class));
        }
        for (StatusConvert statusConvert : StatusConvert.values()) {
            allStatus.get(statusConvert.start).put(statusConvert.target, statusConvert);
        }
    }
    StatusConvert(StatusDefine start, StatusDefine target) {
        this.start = start;
        this.target = target;
    }
    abstract void todoConvertStatus();
    protected void doSomething(Action action){
        System.out.println("转换状态之前的操作");
        action.convertStatus();
        System.out.println("转换状态之后的操作");
    }

}
{% endcodeblock %}

### 测试方法
{% codeblock lang:java %}
public static void main(String[] args) {
    StatusConvert.allStatus.get(StatusDefine.FAILED).get(StatusDefine.SUCCESS).todoConvertStatus();
}
{% endcodeblock %}

> 可以将上面的调用封装成一个静态的执行方法，在此由于方便理解，就不做封装了。

### 测试结果
> 转换状态之前的操作
  失败->成功
  转换状态之后的操作

### 总结

> 以上实现方式能够将所有的状态定义清晰的展现出来，并且所有的状态转换操作都是以StatusDefine(状态定义类)为基础，通过模版方法对状态转换之前和之后的公共代码抽出来，以便我们只关注状态转换的实现。


### 提示
> 以上是博主第一次实现状态机，如有瑕疵还需大家指正。






