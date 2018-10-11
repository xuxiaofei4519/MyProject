---
title: IdentityHashmap解析
date: 2018-04-30 17:10:06
copyright: true
tags:
 - java
categories:
 - java
---

{% cq %}
本篇文章主要浅析IdentityHashmap的使用、适用场景和内部实现原理。
{% endcq %}

<!-- more -->


### **IdentityHashMap 继承类与实现接口**

{% qnimg /java-IdentityHashmap/java-IdentityHashmap01.png %}
### **IdentityHashMap 内部的方法**

{% qnimg /java-IdentityHashmap/java-IdentityHashmap02.png %}


### **IdentityHashMap示例**

- 示例
{% codeblock lang:java %}
@Test
public void testIdentityHashMap(){
   String xanderXu = new String("XanderXu");
   
   Map<String, Object> identityHashMap = new IdentityHashMap<>();
   identityHashMap.put(new String("XanderXu"),"666");
   identityHashMap.put(new String("XanderXu"),"777");
   identityHashMap.put(xanderXu, "xiaofei");
   identityHashMap.put(xanderXu, "xiaofei--2");

   System.out.println(identityHashMap);
}
{% endcodeblock %}

- 运行结果
{% codeblock lang:java %}
{XanderXu=666, XanderXu=xiaofei--2, XanderXu=777}
{% endcodeblock %}
> 从IdentityHashMap的继承关系可以看出IdentityHashMap并非继承于HashMap,而是兄弟关系，共同继承Map，从示例中我们也可以看出IdentitHashMap与HashMap的一大不同：IdentitHashMap允许"equals"为true的key同时存在,但不允许"=="为true的key同时存在。

### **IdentityHashMap 成员变量**
{% codeblock lang:java %}

/**
 * 默认容量
 */
private static final int DEFAULT_CAPACITY = 32;

/**
 * 最小容量
 */
private static final int MINIMUM_CAPACITY = 4;

/**
 * 最大容量
 */
private static final int MAXIMUM_CAPACITY = 1 << 29;

/**
 * 实际存放元素数组
 */
transient Object[] table; // non-private to simplify nested class access

/**
 * 元素个数
 */
int size;

/**
 * 修改次数,以支持快速失败
 */
transient int modCount;

/**
 * NULL对象
 */
static final Object NULL_KEY = new Object();
{% endcodeblock %}
> 从成员变量中可以看出IdentityHashMap数据结构就是一个Object数组,默认容量为32(这里是指存放的键值对数，后面init方法会有讲解),支持迭代器的快速失败,并且对null进行包装(区别put的是null还是原本是null)。


### **核心方法解析**

- init方法
{% codeblock lang:java %}
private void init(int initCapacity) {
    // assert (initCapacity & -initCapacity) == initCapacity; // power of 2
    // assert initCapacity >= MINIMUM_CAPACITY;
    // assert initCapacity <= MAXIMUM_CAPACITY;    
    table = new Object[2 * initCapacity];
}
{% endcodeblock %}
> 因为IdentityHashMap中key和value都是存放数组(table)中的,因此默认容量是32，但是占用的空间是64。所以初始化时要乘以2。


- hash方法
{% codeblock lang:java %}
private static int hash(Object x, int length) {
    int h = System.identityHashCode(x);
    // Multiply by -127, and left-shift to use least bit as part of hash
    return ((h << 1) - (h << 8)) & (length - 1);
}
{% endcodeblock %}
> identityHashCode是一个Native方法,是根据对象的内存地址来计算hash值的。并且length一定是2的n次方,所以减1后和任何数相与得到的永远是偶数，所以key一定是存放在偶数位

- nextKeyIndex方法
{% codeblock lang:java %}
private static int nextKeyIndex(int i, int len) {
    return (i + 2 < len ? i + 2 : 0);
}
{% endcodeblock %}
> 获取下一个key的数组下标

- resize方法
{% codeblock lang:java %}
private boolean resize(int newCapacity) {
    // assert (newCapacity & -newCapacity) == newCapacity; // power of 2
    int newLength = newCapacity * 2;

    Object[] oldTable = table;
    int oldLength = oldTable.length;
    if (oldLength == 2 * MAXIMUM_CAPACITY) { // can't expand any further
        if (size == MAXIMUM_CAPACITY - 1)
            throw new IllegalStateException("Capacity exhausted.");
        return false;
    }
    if (oldLength >= newLength)
        return false;

    Object[] newTable = new Object[newLength];

    for (int j = 0; j < oldLength; j += 2) {
        Object key = oldTable[j];
        if (key != null) {
            Object value = oldTable[j+1];
            oldTable[j] = null;
            oldTable[j+1] = null;
            int i = hash(key, newLength);
            while (newTable[i] != null)
                i = nextKeyIndex(i, newLength);
            newTable[i] = key;
            newTable[i + 1] = value;
        }
    }
    table = newTable;
    return true;
}
{% endcodeblock %}
> 判断容量是否超过最大值,将旧值置空，重新计算hash值，赋值到新table中


- put方法
{% codeblock lang:java %}
public V put(K key, V value) {
    final Object k = maskNull(key);

    retryAfterResize: for (;;) {
        final Object[] tab = table;
        final int len = tab.length;
        int i = hash(k, len);

        for (Object item; (item = tab[i]) != null;
             i = nextKeyIndex(i, len)) {
            if (item == k) {
                @SuppressWarnings("unchecked")
                    V oldValue = (V) tab[i + 1];
                tab[i + 1] = value;
                return oldValue;
            }
        }

        final int s = size + 1;
        // Use optimized form of 3 * s.
        // Next capacity is len, 2 * current capacity.
        if (s + (s << 1) > len && resize(len))
            continue retryAfterResize;

        modCount++;
        tab[i] = k;
        tab[i + 1] = value;
        size = s;
        return null;
    }
}
{% endcodeblock %}
> 先根据hash值获取数组中的位置，然后往后判断是否存在和引用相等的key，如果存在则替换value，返回旧值。终止条件为当前key是null(注意并不是NULL_KEY)
> 如果没有找到相等引用，那么就停止循环，循环后table[i]一定是null的(循环过程中i一直在变)，这时进行插入操作，在插入操作之前判断是否需要扩容。
> 上面s + (s << 1) = s + (s * 2) = s(1=2) = 3s > len,也就是说当键值对的个数大于表长的三分之一的时候就会进行扩容

- get方法
{% codeblock lang:java %}
@SuppressWarnings("unchecked")
public V get(Object key) {
    Object k = maskNull(key);
    Object[] tab = table;
    int len = tab.length;
    int i = hash(k, len);
    while (true) {
        Object item = tab[i];
        if (item == k)
            return (V) tab[i + 1];
        if (item == null)
            return null;
        i = nextKeyIndex(i, len);
    }
}
{% endcodeblock %}
> 先检查key是否是null,然后对key哈希取值确定索引，如果没有找到，就到下一个key，直到为key为null为止。



- closeDeletion方法
{% codeblock lang:java %}
private void closeDeletion(int d) {
    // Adapted from Knuth Section 6.4 Algorithm R
    Object[] tab = table;
    int len = tab.length;

    // Look for items to swap into newly vacated slot
    // starting at index immediately following deletion,
    // and continuing until a null slot is seen, indicating
    // the end of a run of possibly-colliding keys.
    Object item;
    for (int i = nextKeyIndex(d, len); (item = tab[i]) != null;
         i = nextKeyIndex(i, len) ) {
        // The following test triggers if the item at slot i (which
        // hashes to be at slot r) should take the spot vacated by d.
        // If so, we swap it in, and then continue with d now at the
        // newly vacated i.  This process will terminate when we hit
        // the null slot at the end of this run.
        // The test is messy because we are using a circular table.
        int r = hash(item, len);
        if ((i < r && (r <= d || d <= i)) || (r <= d && d <= i)) {
            tab[d] = item;
            tab[d + 1] = tab[i + 1];
            tab[i] = null;
            tab[i + 1] = null;
            d = i;
        }
    }
}   
{% endcodeblock %}
> 删除后,将后面的键值对向前调整,防止找不到key的情况发生。if条件判断该key是否发生hash碰撞，只要发生过碰撞,就会往前移动。



- remove方法
{% codeblock lang:java %}
public V remove(Object key) {
    Object k = maskNull(key);
    Object[] tab = table;
    int len = tab.length;
    int i = hash(k, len);

    while (true) {
        Object item = tab[i];
        if (item == k) {
            modCount++;
            size--;
            @SuppressWarnings("unchecked")
                V oldValue = (V) tab[i + 1];
            tab[i + 1] = null;
            tab[i] = null;
            closeDeletion(i);
            return oldValue;
        }
        if (item == null)
            return null;
        i = nextKeyIndex(i, len);
    }
}
{% endcodeblock %}
> 获取对应key然后hash取值,将对应key删除,并且通过closeDeletion方法对后面的键值对向前做调整



### **总结**

> 1、IdentityHashMap 是通过引用来判断键是否相等的，并且允许null值和null键、允许重复键的Map容器。 
  2、IdentityHashMap 解决哈希冲突的方式是采用线性探测法即往后寻找为null的槽位
  3、IdentityHashMap 默认的初始容量为 32 ，扩容每次扩为原来的两倍。
  4、IdentityHashMap 每一次做删除操作都会调整一次map
