---
title: AQS实现之ReentrantLock的公平锁模式
date: 2018-10-15 20:40:09
copyright: true
tags:
 - java
categories:
 - java
---
{% cq %} 
之前阅读javadoop的文章，根据自己的理解对ReentrantLock进行解析
{% endcq %}
<!-- more -->

#### 流程图：
{% qnimg /java-AQS-01/AQS_01.jpg %}
* * *

#### 代码详解

##### ReentrantLock构造方法
```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```
- 从上面的构造函数可以看出可重入锁ReentrantLock有两种模式。

##### ReentrantLock公平锁模式时的内部类
```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```
##### AQS中的acquire方法
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```