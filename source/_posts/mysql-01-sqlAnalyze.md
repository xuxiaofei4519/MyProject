---
title: 丁奇mysql01：sql相关
date: 2019-01-17 17:12:00
copyright: true
tags:
 - mysql
categories:
 - mysql
---
{% cq %} 
最近在学习丁奇系列的干货，收获颇多，因此自己手动实践，并总结一些工作中会经常用到的知识，这一篇总结数据库连接、sql分析相关内容。
{% endcq %}
<!-- more -->

#### 查看数据库连接列表
```
show processlist;
```
{% qnimg /mysql-01-sqlAnalyze/mysql01-01.png %}

可以查看该数据库的所有连接，以及他们的状态

#### sql执行时间分析
```sql
select @@profiling;
```
{% qnimg /mysql-01-sqlAnalyze/mysql01-02.png %}

```sql
set profiling=1;
```
{% qnimg /mysql-01-sqlAnalyze/mysql01-03.png %}

执行查询语句

{% qnimg /mysql-01-sqlAnalyze/mysql01-04.png %}

查看执行时间

{% qnimg /mysql-01-sqlAnalyze/mysql01-05.png %}

查看 SQL 执行耗时详细信息
```
show profile for query [id]
```

{% qnimg /mysql-01-sqlAnalyze/mysql01-06.png %}

以上具体的信息都是从 INFORMATION_SCHEMA.PROFILING 这张表中取得的。这张表记录了所有的各个步骤的执行时间及相关信息。可以查询指定queryid的sql执行，从开始、分析器、优化器、执行器到最终结束的详细信息
```sql
select * from INFORMATION_SCHEMA.PROFILING where query_id = Query_ID;
```

{% qnimg /mysql-01-sqlAnalyze/mysql01-07.png %}

#### 重新初始化连接资源

```
mysql_reset_connection
```
 
这并非是sql命令，其实是mysql的API接口，通过编程可以调用。

#### 查询缓存
查询缓存配置
mac机器需要在/etc/目录下新建my.cnf文件
文件内容如下:

{% qnimg /mysql-01-sqlAnalyze/mysql01-08.png %}

query_cache_type=0时表示关闭，1时表示打开，2表示只要select中明确指定SQL_CACHE才缓存。


我们可以通过以下命令查看当前查询缓存的设置
```sql
show variables like ‘%query_cache%';
```
{% qnimg /mysql-01-sqlAnalyze/mysql01-09.png %}

发现query_cache_type为ON代表开启了


如果我们开启了查询缓存，那么可以通过下面的命令查看缓存的相关信息
```sql
show status like ‘%Qcache%';
```

{% qnimg /mysql-01-sqlAnalyze/mysql01-10.png %}

以上几个参数的含义：

* Qcache_free_blocks:表示查询缓存中目前还有多少剩余的blocks，如果该值显示较大，则说明查询缓存中的内存碎片过多了，可能在一定的时间进行整理。
* Qcache_free_memory:查询缓存的内存大小，通过这个参数可以很清晰的知道当前系统的查询内存是否够用，是多了，还是不够用，DBA可以根据实际情况做出调整。
* Qcache_hits:表示有多少次命中缓存。我们主要可以通过该值来验证我们的查询缓存的效果。数字越大，缓存效果越理想。
* Qcache_inserts: 表示多少次未命中然后插入，意思是新来的SQL请求在缓存中未找到，不得不执行查询处理，执行查询处理后把结果insert到查询缓存中。这样的情况的次数，次数越多，表示查询缓存应用到的比较少，效果也就不理想。当然系统刚启动后，查询缓存是空的，这很正常。
* Qcache_lowmem_prunes:该参数记录有多少条查询因为内存不足而被移除出查询缓存。通过这个值，用户可以适当的调整缓存大小。
* Qcache_not_cached: 表示因为query_cache_type的设置而没有被缓存的查询数量。
* Qcache_queries_in_cache:当前缓存中缓存的查询数量。
* Qcache_total_blocks:当前缓存的block数量。

查询缓存的一些问题：
1. 把SQL语句的hash值作为键，SQL语句的结果集作为值；这样就引起了一个问题如`select user from mysql.user` 和 `SELECT user FROM mysql.user` 这两个将会被当成不同的SQL语句，这个时候就算结果集已经有了，但是一然用不到。
2. 当查询所基于的低层表有改动时与这个表有关的查询缓存都会作废、如果对于并发度比较大的系统这个开销是可观的；对于作废结果集这个操作也是要用并发访问控制的，就是说也会有锁。并发大的时候就会有Waiting for query cache lock 产生。
3. 至于用不用还是要看业务需求。

#### 慢日志
1.开启慢查询日志

{% qnimg /mysql-01-sqlAnalyze/mysql01-11.png %}

2.设置超时时间,默认是10秒，我们可以通过命令设置：

{% qnimg /mysql-01-sqlAnalyze/mysql01-12.png %}

然后新开一个会话查看：

{% qnimg /mysql-01-sqlAnalyze/mysql01-13.png %}

3.log_output
参数是指定日志的存储方式。log_output=‘FILE’表示将日志存入文件，默认值是’FILE’。log_output='TABLE’表示将日志存入数据库，这样日志信息就会被写入到mysql.slow_log表中。MySQL数据库支持同时两种日志存储方式，配置的时候以逗号隔开即可，如：log_output=‘FILE,TABLE’。日志记录到系统的专用日志表中，要比记录到文件耗费更多的系统资源，因此对于需要启用慢查询日志，又需要能够获得更高的系统性能，那么建议优先记录到文件。
4.log-queries-not-using-indexes
未使用索引的查询也被记录到慢查询日志中（可选项）。如果调优的话，建议开启这个选项。另外，开启了这个参数，其实使用full index scan的sql也会被记录到慢查询日志。
5.log_slow_admin_statements
表示是否将慢管理语句例如ANALYZE TABLE和ALTER TABLE等记入慢查询日志

#### 慢日志分析工具
MySQL 提供了慢日志分析工具 mysqldumpslow。

-s 表示按照何种方式排序；
c: 访问计数
l: 锁定时间
r: 返回记录
t: 查询时间
al:平均锁定时间
ar:平均返回记录数
at:平均查询时间
-t 是top n的意思，即为返回前面多少条的数据；
-g 后边可以写一个正则匹配模式，大小写不敏感的；

命令示例：
得到返回记录集最多的 10 个 SQL：mysqldumpslow -s r -t 10 /database/mysql/mysql06_slow.log
得到访问次数最多的 10 个 SQL：mysqldumpslow -s c -t 10 /database/mysql/mysql06_slow.log
得到按照时间排序的前10条里面含有左连接的查询语句：mysqldumpslow -s t -t 10 -g “left join” /database/mysql/mysql06_slow.log
另外建议在使用这些命令时结合 | 和 more 使用 ，否则有可能出现刷屏的情况：mysqldumpslow -s r -t 20 /mysqldata/mysql/mysql06-slow.log | more