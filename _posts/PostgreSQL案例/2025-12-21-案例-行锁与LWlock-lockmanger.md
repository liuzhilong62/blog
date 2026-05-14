---
layout: post
title: "案例-行锁与LWlock-lockmanger"
date: 2025-12-21
categories: [PostgreSQL案例]
---

# 现象

数据库有大量行锁和少部分LWlock lockmanger，cpu打满，活动会话飙升。锁对应的block pid在变化，未见明显长事务阻塞。
(脑补cpu和活动会话高）

大量的锁对于的sql如下：

```sql
UPDATE lzl_record  SET rc_lzl1= rc_lzl1 + $1, pc_lzl2 = pc_lzl2 + $2, rc_lzl3 = rc_lzl3 + $3 where lzl_id = $4
```



# 分析

## SQL未见并发上涨

从hit和cpu的关联性看，可以从sql的hit分析。其中那个UPDATE sql占比80%左右，该sql的执行次数没有变化，但blks hit明显异常。

同时分析元数据访问，快照内均元数据表没有访问特别高的。

从现象分析看不出来SQL并发上涨，也看不出来元数据异常。SQL hit上涨暂时看不出来为什么。



## LWlock lockmanager分析

因为SQL本身简单，lzl_record表的lzl_id字段是唯一字段，也就是通过唯一键进行更新。

现场的等待事件除了大量的显示锁以外，还有LWlock lockmanager。

但表是普通表不是分区表，且表上只有4、5个索引。

LWlock lockmanger跟没有用到fast path有关系，而简单的查询、DML可以用到fastpath

> Weak relation locks.  SELECT, INSERT, UPDATE, and DELETE must acquire a
> lock on every relation they operate on, as well as various system catalogs
> that can be used internally.  Many DML operations can proceed in parallel
> against the same table at the same time; only DDL operations such as
> CLUSTER, ALTER TABLE, or DROP -- or explicit user action such as LOCK TABLE
> -- will create lock conflicts with the "weak" locks (AccessShareLock,
> RowShareLock, RowExclusiveLock) acquired by DML operations.



所以，一个不超过访问16个relation（含索引）的SELECT/DML是可以用到fastpath的，就不应该有很多LWlock lockmanager。

但是，DML肯定不能简单的用fastpath，fastpath是完全本地化处理锁，DML肯定需要校验其他会话是否持有行，需要访问共享内存。再加上SQL通过唯一字段更新还能有行锁，说明肯定更新的是同一行。

从日志上可以看到更新同一行的情况，其中一行有锁等待的更新有上万次。

# 压测

## 压测同一行更新产生LWLock LockManager

出于行锁肯定不能仅fastpath的考虑，以及LWLock LockManager会降低数据库性能的经验，压测不同场景性能情况。

```
#prompt
给我一个pgbench压测脚本
表结构：主键,唯一字段+唯一索引，其他字段
更新：通过唯一字段更新

压测重复更新同一行数据，也就是重复行锁更新
压测随机更新某一行数据，也就是无行锁更新
```

脚本略，套餐20c96g

pgbench命令如下

```shell
pgbench -h localhost -p $PGPORT -d lzldb -U dbmgr -f update_same_unique_key.sql  -c 200 -j 32 -T 600 -r -S
pgbench -h localhost -p $PGPORT -d lzldb -U dbmgr -f update_random_unique_key.sql  -c 200 -j 32 -T 600 -r -S
```

压测时的等待事件：

```sql
 --更新同一行，常见2次等待
 usename  | state  |     wait_event      | wait_event_type | cnt 
----------+--------+---------------------+-----------------+-----
 dbmgr    | active | LockManager         | LWLock          | 105
 dbmgr    | active | transactionid       | Lock            |  61
 dbmgr    | active | tuple               | Lock            |  25
 dbmgr    | active | [null]              | [null]          |   8
 dbmgr    | active | WALSync             | IO              |   1
 
  usename  | state  |     wait_event      | wait_event_type | cnt 
----------+--------+---------------------+-----------------+-----
 dbmgr    | active | transactionid       | Lock            | 180
 dbmgr    | active | LockManager         | LWLock          |  18
 dbmgr    | active | tuple               | Lock            |   1
 dbmgr    | active | WALSync             | IO              |   1
```

```sql
--更新不同行，常见2次等待
usename  |        state        |     wait_event      | wait_event_type | cnt 
----------+---------------------+---------------------+-----------------+-----
 dbmgr    | active              | [null]              | [null]          | 106
 dbmgr    | idle                | ClientRead          | Client          |  34
 dbmgr    | idle in transaction | ClientRead          | Client          |  25
 dbmgr    | active              | WALWrite            | LWLock          |  21
 dbmgr    | active              | BufferMapping       | LWLock          |   7
 dbmgr    | idle in transaction | [null]              | [null]          |   4
 dbmgr    | idle in transaction | WALWrite            | LWLock          |   2
 
  usename  |        state        |     wait_event      | wait_event_type | cnt 
----------+---------------------+---------------------+-----------------+-----
 dbmgr    | active              | [null]              | [null]          | 117
 dbmgr    | idle                | ClientRead          | Client          |  42
 dbmgr    | idle in transaction | ClientRead          | Client          |  24
 dbmgr    | active              | WALWrite            | LWLock          |  12
 dbmgr    | active              | XactGroupUpdate     | IPC             |   1
 dbmgr    | active              | WALSync             | IO              |   1
 dbmgr    | active              | XactSLRU            | LWLock          |   1
 dbmgr    | active              | BufferContent       | LWLock          |   1
 dbmgr    | active              | ClientRead          | Client          |   1
```

从等待事件上可以看出差异，更新同一行会出现LWLock LockManager甚至占比较高的情况，不同行基本都是等cpu。场景1与生产情况类似。

# 行锁与fastpath浅析

lmgr readme对fastpath的解释：

```
Fast Path Locking
-----------------

Fast path locking is a special purpose mechanism designed to reduce the
overhead of taking and releasing certain types of locks which are taken
and released very frequently but rarely conflict.  Currently, this includes
two categories of locks:

(1) Weak relation locks.  SELECT, INSERT, UPDATE, and DELETE must acquire a
lock on every relation they operate on, as well as various system catalogs
that can be used internally.  Many DML operations can proceed in parallel
against the same table at the same time; only DDL operations such as
CLUSTER, ALTER TABLE, or DROP -- or explicit user action such as LOCK TABLE
-- will create lock conflicts with the "weak" locks (AccessShareLock,
RowShareLock, RowExclusiveLock) acquired by DML operations.
```

能使用fastpath的锁的条件`lmgr/lock.c`：

```c
/*
 * The fast-path lock mechanism is concerned only with relation locks on
 * unshared relations by backends bound to a database.  The fast-path
 * mechanism exists mostly to accelerate acquisition and release of locks
 * that rarely conflict.  Because ShareUpdateExclusiveLock is
 * self-conflicting, it can't use the fast-path mechanism; but it also does
 * not conflict with any of the locks that do, so we can ignore it completely.
 */
#define EligibleForRelationFastPath(locktag, mode) \
	((locktag)->locktag_lockmethodid == DEFAULT_LOCKMETHOD && \
	(locktag)->locktag_type == LOCKTAG_RELATION && \
	(locktag)->locktag_field1 == MyDatabaseId && \
	MyDatabaseId != InvalidOid && \
	(mode) < ShareUpdateExclusiveLock)
```

SELECT/DML是可以使用fastpath的，但是仅限于locktype=relation。

实际查看行锁时的锁情况：

```sql
--窗口1
begin;
update lzl1 set b='zzz' where a=1;

--窗口2
begin;
update lzl1 set b='zzz' where a=1;
--waiting


--窗口3
 select *  from pg_locks where pid<>(select pg_backend_pid()) order by pid,locktype;
   locktype    | database | relation |  page  | tuple  | virtualxid | transactionid | classid | objid  | objsubid | virtualtransaction |  pid   |       mode       | granted | fastpath 
---------------+----------+----------+--------+--------+------------+---------------+---------+--------+----------+--------------------+--------+------------------+---------+----------
 relation      |  4267681 |  5290151 | [null] | [null] | [null]     |        [null] |  [null] | [null] |   [null] | 5/4791             | 220559 | RowExclusiveLock | t       | t
 transactionid |   [null] |   [null] | [null] | [null] | [null]     |     170706189 |  [null] | [null] |   [null] | 5/4791             | 220559 | ExclusiveLock    | t       | f
 transactionid |   [null] |   [null] | [null] | [null] | [null]     |     170706190 |  [null] | [null] |   [null] | 5/4791             | 220559 | ExclusiveLock    | t       | f
 transactionid |   [null] |   [null] | [null] | [null] | [null]     |     170706187 |  [null] | [null] |   [null] | 5/4791             | 220559 | ShareLock        | f       | f
 tuple         |  4267681 |  5290151 |      0 |      1 | [null]     |        [null] |  [null] | [null] |   [null] | 5/4791             | 220559 | ExclusiveLock    | t       | f
 virtualxid    |   [null] |   [null] | [null] | [null] | 5/4791     |        [null] |  [null] | [null] |   [null] | 5/4791             | 220559 | ExclusiveLock    | t       | t
 relation      |  4267681 |  5290151 | [null] | [null] | [null]     |        [null] |  [null] | [null] |   [null] | 7/562              | 253641 | RowExclusiveLock | t       | t
 transactionid |   [null] |   [null] | [null] | [null] | [null]     |     170706187 |  [null] | [null] |   [null] | 7/562              | 253641 | ExclusiveLock    | t       | f
 virtualxid    |   [null] |   [null] | [null] | [null] | 7/562      |        [null] |  [null] | [null] |   [null] | 7/562              | 253641 | ExclusiveLock    | t       | t

```

pg的行锁实现比较复杂，不仅有tuple锁，还需要transactionid和relation锁。其中只有locktype=relation、virtualxid能用到fastpath，其他都用不到。

对比没有行锁：

```sql
--窗口1
begin;
update lzl1 set b='zzz' where a=1;

--窗口2
begin;
update lzl1 set b='zzz' where a=2;
--waiting

select *  from pg_locks where pid<>(select pg_backend_pid()) order by pid,locktype;
   locktype    | database | relation |  page  | tuple  | virtualxid | transactionid | classid | objid  | objsubid | virtualtransaction |  pid   |       mode       | granted | fastpath 
---------------+----------+----------+--------+--------+------------+---------------+---------+--------+----------+--------------------+--------+------------------+---------+----------
 relation      |  4267681 |  5290151 | [null] | [null] | [null]     |        [null] |  [null] | [null] |   [null] | 5/4792             | 220559 | RowExclusiveLock | t       | t
 relation      |  4267681 |  5290151 | [null] | [null] | [null]     |        [null] |  [null] | [null] |   [null] | 5/4792             | 220559 | AccessShareLock  | t       | t
 transactionid |   [null] |   [null] | [null] | [null] | [null]     |     170706214 |  [null] | [null] |   [null] | 5/4792             | 220559 | ExclusiveLock    | t       | f
 virtualxid    |   [null] |   [null] | [null] | [null] | 5/4792     |        [null] |  [null] | [null] |   [null] | 5/4792             | 220559 | ExclusiveLock    | t       | t
 relation      |  4267681 |  5290151 | [null] | [null] | [null]     |        [null] |  [null] | [null] |   [null] | 7/563              | 253641 | AccessShareLock  | t       | t
 relation      |  4267681 |  5290151 | [null] | [null] | [null]     |        [null] |  [null] | [null] |   [null] | 7/563              | 253641 | RowExclusiveLock | t       | t
 transactionid |   [null] |   [null] | [null] | [null] | [null]     |     170706212 |  [null] | [null] |   [null] | 7/563              | 253641 | ExclusiveLock    | t       | f
 virtualxid    |   [null] |   [null] | [null] | [null] | 7/563      |        [null] |  [null] | [null] |   [null] | 7/563              | 253641 | ExclusiveLock    | t       | t

```

fastpath=f的也就少了2、3个。2个会话持有的transactionid锁是肯定用不到fastpath了。

总结使用fastpath锁机制的场景（且）：

- 锁等级<=3，即SELECT/DML语句
- locktype=relation。pg的行锁还至少需要transactionid和tuple锁，所以这俩用不到fastpath
- relation数少于16个（一般分区表全分区访问就超了）



# 总结

1.行锁是因是果，是行锁的问题还是数据库性能下降SQL跑的慢了出现行锁？

行锁是因。sql执行次数没有变化，但是sql的传参从离散变成集中，即更新同一行的情况明显增加。从压测数据来看更新同一行会出现行锁和LWLock LockManager的等待。

2.sql执行次数没有上涨，SQL性能是否下降？

SQL性能其实是下降了，但是索引肯定没有走错，只是因为反复更新了同一行。



解决方案：

从业务层了解SQL是某接口开关调用后讲调用次数更新到表里。如果是同一接口反复调用，是可能出现反复更新同一行的情况的。所以，减少同一接口反复调用或者更大批次的更新数据库，预计可以减缓这个问题。