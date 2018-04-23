---
title: EnumMap浅析
date: 2018-04-19 14:40:09
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

以上成员变量含义:keyType表示类型信息，keyUniverse表示键，是所有可能的枚举值，vals表示键对应的值，size表示键值对个数。



### 构造方法


{% codeblock lang:java %}
/**
 * 通过枚举的Class类型构造
 */
public EnumMap(Class<K> keyType) {
    this.keyType = keyType;
    keyUniverse = getKeyUniverse(keyType);
    vals = new Object[keyUniverse.length];
}

/**
 * 通过EnumMap构造
 */
public EnumMap(EnumMap<K, ? extends V> m) {
     keyType = m.keyType;
     keyUniverse = m.keyUniverse;
     vals = m.vals.clone();
     size = m.size;
}

/**
 * 通过Map构造,必须保证key的类型是枚举类型
 */
public EnumMap(Map<K, ? extends V> m) {
    if (m instanceof EnumMap) {
        EnumMap<K, ? extends V> em = (EnumMap<K, ? extends V>) m;
        keyType = em.keyType;
        keyUniverse = em.keyUniverse;
        vals = em.vals.clone();
        size = em.size;
    } else {
        if (m.isEmpty())
            throw new IllegalArgumentException("Specified map is empty");
        keyType = m.keySet().iterator().next().getDeclaringClass();
        keyUniverse = getKeyUniverse(keyType);
        vals = new Object[keyUniverse.length];
        putAll(m);
    }
}
{% endcodeblock %}


### put方法解析
{% codeblock lang:java %}
public V put(K key, V value) {
    typeCheck(key);
    int index = key.ordinal();
    Object oldValue = vals[index];
    vals[index] = maskNull(value);
    if (oldValue == null)
        size++;
    return unmaskNull(oldValue);
}

private void typeCheck(K key) {
    Class keyClass = key.getClass();
    if (keyClass != keyType && keyClass.getSuperclass() != keyType)
        throw new ClassCastException(keyClass + " != " + keyType);
}


private Object maskNull(Object value) {
    return (value == null ? NULL : value);
}

private V unmaskNull(Object value) {
    return (V) (value == NULL ? null : value);
}

private static final Object NULL = new Object() {
    public int hashCode() {
        return 0;
    }
    public String toString() {
        return "java.util.EnumMap.NULL";
    }
};

{% endcodeblock %}


- typeCheck方法校验传入key的类型是否与EnumMap初始化时定义的枚举类型对应(包含子类)
- EnumMap允许value为null，为了区别null值与没有值，EnumMap将null值包装成了一个特殊的NULL对象，
  有两个辅助方法用于null的打包和解包，打包方法为maskNull，解包方法为unmaskNull。
- ordinal()方法是获取key在枚举类中的顺序(索引),将该索引作为往vals数组存放value的索引
- 当put相同的key不同value的时候.会重新将旧value覆盖，然后将value值置空。



### get方法解析
{% codeblock lang:java %}
public V get(Object key) {
    return (isValidKey(key) ?
            unmaskNull(vals[((Enum)key).ordinal()]) : null);
}

private boolean isValidKey(Object key) {
    if (key == null)
        return false;
    // Cheaper than instanceof Enum followed by getDeclaringClass
    Class keyClass = key.getClass();
    return keyClass == keyType || keyClass.getSuperclass() == keyType;
}

{% endcodeblock %}


- 从上面代码可以看出，key通过数组下标映射数据，因此get数据的时候速度效率非常高。

### remove方法解析

{% codeblock lang:java %}
public V remove(Object key) {
    if (!isValidKey(key))
        return null;
    int index = ((Enum<?>)key).ordinal();
    Object oldValue = vals[index];
    vals[index] = null;
    if (oldValue != null)
        size--;
    return unmaskNull(oldValue);
}
{% endcodeblock %}

- 从上面代码可以看出,删除元素之前先对key进行验证，如果key不是map初始化时指定的枚举类型，那么将会返回null。
- 当验证完key之后，再用该key在枚举类中的顺序号作为寻找value的下标，通过该下标将val数组中的值置空。如果value不为null的话，map的size就减1。
- 最终将卸载NULL对象，返回删除的value。在remove操作中是不会删除key数组(keyUniverse[])中的任何元素。keyUniverse[]在类构造阶段已经初始化完毕，一直伴随着map的整个生命周期，直到该EnumMap被卸载。
- 这里有一个问题需要注意，从代码中我们没有看到任何关于线程安全的代码，因此就会产生ABA的问题。当多线程环境下，很有可能发生当我们删除一个key的value时，value置空了，但是size--没有执行，这时候又有一个线程对相同的key进行put操作,我们获取的size大小就有可能不变,也有可能变大。产生脏读的情况。



### 使用方法

{% codeblock lang:java %}
import java.util.EnumMap;

public class Main {
    public static void main(String[] args) {
        EnumMap<ResultEnum, String> map = new EnumMap<ResultEnum, String>(ResultEnum.class);
        map.put(ResultEnum.SUCCESS,"1");
        map.put(ResultEnum.SUCCESS,"2");
        map.put(ResultEnum.FAIL,"3");
        map.put(ResultEnum.DEALING,"4");
        map.put(ResultEnum.UNKNOWN,"5");
        System.out.println(map);
    }
}
{% endcodeblock %}

输出结果:
{% codeblock lang:java %}
{SUCCESS=2, FAIL=3, DEALING=4, UNKNOWN=5}
{% endcodeblock %}


###  使用场景
状态机：根据老大的提示，周六在公司回顾EnumMap的时候，发现确实可以使用EmumMap作为状态机。状态机实现见：EnumMap状态机实现。

分类场景：对于自己而言，更多的是数据分类场景，同一类的数据对应到同一个枚举，即枚举对应Map的形式。

由于EnumMap并非线程安全，因此并不适合并发修改的场景。当然，也可以自定义将EnumMap封装成适用并发的Map类，这些都是后话了。
  
### 总结

> 以上就是EnumMap的基本实现原理，内部有两个数组，长度相同，一个表示所有可能的键，一个表示对应的值，值为null表示没有该键值对，键都有一个对应的索引，根据索引可直接访问和操作其键和值，效率很高。
  EnumMap的缺点就是并非是线程安全的，可以用工具类包装成现成安全的:Map<EnumKey, V> m = Collections.synchronizedMap(new EnumMap<EnumKey, V>();

