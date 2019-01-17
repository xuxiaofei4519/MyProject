---
title: 丁奇mysql03：长事务查询
date: 2019-01-17 10:45:00
copyright: true
tags:
 - mysql
categories:
 - mysql
---

{% cq %} 
最近在学习丁奇系列的干货，收获颇多，因此自己手动实践，并总结一些工作中会经常用到的知识，这一篇总结如何查询长事务
{% endcq %}
<!-- more -->

#### 查询长事务

查询长事务的sql命令：
```sql
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```

#### 模拟过程

1. 关闭自动提交
{% qnimg /mysql-03-longTransaction/mysql03-01.png %}
2. 开始事务
{% qnimg /mysql-03-longTransaction/mysql03-02.png %}
3. 执行sql
{% qnimg /mysql-03-longTransaction/mysql03-03.png %}
4. 执行查询命令
{% qnimg /mysql-03-longTransaction/mysql03-04.png %}


> 经过以上步骤可以看到我们能够查询出长事务的线程id。同时我在执行时发现，步骤2并没有真正开启事务，因此如果执行完步骤2，去查询长事务是查询不到的，只有真正执行sql语句的时候，事务才会真正开启，这个时候才可以查询出长事务，这也和丁奇文章中所说的对应。
然后还要注意的是1步骤其实可以去掉，自动提交的意思是在没有begin或start transaction的时候，因此就算autocommit=1.我们使用命令start transaction还是可以开始事务的，事务内的语句也不会自动提交。
