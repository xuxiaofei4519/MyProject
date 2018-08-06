---
title: WeakHashMap解析
date: 2018-05-05 17:34:06
copyright: true
tags:
 - java
categories:
 - java
---

{% cq %}
本篇文章主要浅析WeakHashMap的使用、适用场景和内部实现原理。
{% endcq %}

<!-- more -->


### **IdentityHashMap 继承类与实现接口**

![截图](/image/java-WeakHashMap/map01.png)


### **IdentityHashMap 内部的方法**

![截图](/image/java-WeakHashMap/map02.png)


### 成员变量
{% codeblock lang:java %}
/**
 * 默认初始容量
 */
private static final int DEFAULT_INITIAL_CAPACITY = 16;


/**
 * 最大容量
 */
private static final int MAXIMUM_CAPACITY = 1 << 30;


/**
 *  如果构造方法中没有指定默认加载因子 0.75
 */
private static final float DEFAULT_LOAD_FACTOR = 0.75f;


/**
 * 元素Entry数组
 */
Entry<K,V>[] table;


/**
 * 元素个数
 */
private int size;


/**
 * 要调整大小的下一个值(容量*负载系数)。
 */
private int threshold;


/**
 * 哈希表的装载因子
 */
private final float loadFactor;


/**
 * 引用队列,用于存储已经被GC的元素引用
 */
private final ReferenceQueue<Object> queue = new ReferenceQueue<>();


/**
 * 修改计数
 */
int modCount;

{% endcodeblock %}

### **核心方法解析**

- 构造方法
{% codeblock lang:java %}
public WeakHashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Initial Capacity: "+
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;

    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal Load factor: "+
                                           loadFactor);
    int capacity = 1;
    while (capacity < initialCapacity)
        capacity <<= 1;
    table = newTable(capacity);
    this.loadFactor = loadFactor;
    threshold = (int)(capacity * loadFactor);
}
{% endcodeblock %}
>判断是否指定容量大小是否是在正常范围,然后计算容量大小并初始化数组,并给threshold赋值(计算触发下一次扩容时的容量大小)

- put方法
{% codeblock lang:java %}
public V put(K key, V value) {
    Object k = maskNull(key);
    int h = hash(k);
    Entry<K,V>[] tab = getTable();
    int i = indexFor(h, tab.length);

    for (Entry<K,V> e = tab[i]; e != null; e = e.next) {
        if (h == e.hash && eq(k, e.get())) {
            V oldValue = e.value;
            if (value != oldValue)
                e.value = value;
            return oldValue;
        }
    }

    modCount++;
    Entry<K,V> e = tab[i];
    tab[i] = new Entry<>(k, value, queue, h, e);
    if (++size >= threshold)
        resize(tab.length * 2);
    return null;
}
{% endcodeblock %}
>计算hash值,根据hash值计算数组的索引值,遍历Entry链表，查看是否有相等的key存在，相等替换对应值。如果key不相等，添加键值对。如果超过扩容边界值，那么将重新扩容。


- get方法
{% codeblock lang:java %}
public V get(Object key) {
    Object k = maskNull(key);
    int h = hash(k);
    Entry<K,V>[] tab = getTable();
    int index = indexFor(h, tab.length);
    Entry<K,V> e = tab[index];
    while (e != null) {
        if (e.hash == h && eq(k, e.get()))
            return e.value;
        e = e.next;
    }
    return null;
}
{% endcodeblock %}
>原理和put方法类似,根据key的hash值计算数组索引，遍历链表,找到相等key获取value并返回


- remove方法
{% codeblock lang:java %}
public V remove(Object key) {
    Object k = maskNull(key);
    int h = hash(k);
    Entry<K,V>[] tab = getTable();
    int i = indexFor(h, tab.length);
    Entry<K,V> prev = tab[i];
    Entry<K,V> e = prev;

    while (e != null) {
        Entry<K,V> next = e.next;
        if (h == e.hash && eq(k, e.get())) {
            modCount++;
            size--;
            if (prev == e)
                tab[i] = next;
            else
                prev.next = next;
            return e.value;
        }
        prev = e;
        e = next;
    }

    return null;
}
{% endcodeblock %}
> 根据key的hash值计算索引值,然后遍历单向链表判断相等的key,如果key相等将前一个键值对直接指向下一个键值对，这样就过掉了当前的键值对。上面prev==e 是为了判断当key和链表中的第一个Entry就相等的话，直接将tab[i]赋值成下一个元素


- Entry实现
{% codeblock lang:java %}
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
    V value;
    final int hash;
    Entry<K,V> next;

    /**
     * Creates new entry.
     */
    Entry(Object key, V value,
          ReferenceQueue<Object> queue,
          int hash, Entry<K,V> next) {
        super(key, queue);
        this.value = value;
        this.hash  = hash;
        this.next  = next;
    }

    @SuppressWarnings("unchecked")
    public K getKey() {
        return (K) WeakHashMap.unmaskNull(get());
    }

    public V getValue() {
        return value;
    }

    public V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>)o;
        K k1 = getKey();
        Object k2 = e.getKey();
        if (k1 == k2 || (k1 != null && k1.equals(k2))) {
            V v1 = getValue();
            Object v2 = e.getValue();
            if (v1 == v2 || (v1 != null && v1.equals(v2)))
                return true;
        }
        return false;
    }

    public int hashCode() {
        K k = getKey();
        V v = getValue();
        return Objects.hashCode(k) ^ Objects.hashCode(v);
    }

    public String toString() {
        return getKey() + "=" + getValue();
    }
}
{% endcodeblock %}
> 因为WeakHashMap中的Entry都是弱引用,所以一旦weakhashmap外的强引用断掉，那么这个对象就会被回收，weakhashMap指向该对象的引用也会失效。


### **总结**
> WeakHashMap和hashMap相似,内部都是维护了hash表和链表。不同的是WeakHashMap链表中的Entry继承了弱引用，当外部强引用失效后，weakHashMap对应的键值对也会失效。
> 因此WeakHashMap适合存储元素经常发生变化的数据,可以防止发生强引用导致的内存泄漏。


### **注意**
> 常量数据存储在WeakHashMap中，无论其外部key的引用是否为null,都不会在weakhashMap中被清除，只有手动remove才会被清除,例如下面的示例

{% codeblock lang:java %}
@Test
public void testWeakHashMap(){

WeakHashMap weakHashMap = new WeakHashMap();
String a1 = "a1";
String a2 = "a2";
String a3 = new String("a3");
weakHashMap.put(a1 ,"a1");
weakHashMap.put(a2, "a2");
weakHashMap.put(a3, "a3");
System.out.println(weakHashMap);

a3 = null;
System.gc();
System.out.println(weakHashMap);
}
{% endcodeblock %}
> String a1 = "a1" 是存储在常量池中的 即使a1=null, weakHashMap中的a1键值对还是会存在，因为a1引用的指向的内存区域数据还是存在，通俗讲就是a1指向的是常量池，GC不会回收常量池中的内容。所以weakHashMap不会影响指向常量数据的引用。