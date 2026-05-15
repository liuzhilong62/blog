---
layout: post
title: "案例-添加索引性能下降和generic plan"
date: 2025-09-13
categories: [PostgreSQL案例]
description: "分析添加索引后性能反而下降的案例：新建索引导致优化器选择不同执行路径，配合generic plan缓存使analyze无法更新已缓存的错误计划"
---

# 问题现象

前晚添加索引，第二天早上cpu打爆，sql容易定位，问题sql就1条。该sql跑了30多s，但是昨天跑3s左右，所以要看下前后执行计划变化。

执行计划只贴关键部分。

加索引前的执行计划：

```sql
            ->  Nested Loop  (cost=19.92..2259694.20 rows=265822 width=33)
                  ->  Index Scan using uk_lzl_task on lzl_task t  (cost=0.29..20007.99 rows=195 width=24)
                        Filter: ((created_by)::text = 'LIUZHILONG62'::text)
                  ->  Append  (cost=19.63..11337.15 rows=14842 width=57)
                        ->  Bitmap Heap Scan on lzl_202501 cc_1  (cost=19.63..3053.69 rows=1467 width=66)
                              Recheck Cond: ((task_no)::text = (t.task_no)::text)
                              Filter: ((created_date > '2025-01-07 09:00:00'::timestamp without time zone) AND (created_date < '2025-09-03 12:56:44.973'::timestamp without time zone))
                              ->  Bitmap Index Scan on lzl_202501_task_no_idx  (cost=0.00..19.27 rows=1594 width=0)
                                    Index Cond: ((task_no)::text = (t.task_no)::text)
                        ->  Bitmap Heap Scan on lzl_202502 cc_2  (cost=21.67..3066.85 rows=1604 width=66)
                              Recheck Cond: ((task_no)::text = (t.task_no)::text)
                              Filter: ((created_date > '2025-01-07 09:00:00'::timestamp without time zone) AND (created_date < '2025-09-03 12:56:44.973'::timestamp without time zone))
                              ->  Bitmap Index Scan on lzl_202502_task_no_idx  (cost=0.00..21.27 rows=1605 width=0)
                                    Index Cond: ((task_no)::text = (t.task_no)::text)
                        ->  Index Scan using lzl_202503_task_no_idx on lzl_202503 cc_3  (cost=0.43..1362.61 rows=1637 width=57)
                              Index Cond: ((task_no)::text = (t.task_no)::text)
                              Filter: ((created_date > '2025-01-07 09:00:00'::timestamp without time zone) AND (created_date < '2025-09-03 12:56:44.973'::timestamp without time zone))
                        ->  Index Scan using lzl_202504_task_no_idx on lzl_202504 cc_4  (cost=0.43..604.64 rows=1795 width=56)
                              Index Cond: ((task_no)::text = (t.task_no)::text)
                              Filter: ((created_date > '2025-01-07 09:00:00'::timestamp without time zone) AND (created_date < '2025-09-03 12:56:44.973'::timestamp without time zone))
                        ->  Index Scan using lzl_202505_task_no_idx on lzl_202505 cc_5  (cost=0.43..445.30 rows=1450 width=56)
                              Index Cond: ((task_no)::text = (t.task_no)::text)
                              Filter: ((created_date > '2025-01-07 09:00:00'::timestamp without time zone) AND (created_date < '2025-09-03 12:56:44.973'::timestamp without time zone))
                        ->  Index Scan using lzl_202506_task_no_idx on lzl_202506 cc_6  (cost=0.43..583.94 rows=1675 width=56)
                              Index Cond: ((task_no)::text = (t.task_no)::text)
                              Filter: ((created_date > '2025-01-07 09:00:00'::timestamp without time zone) AND (created_date < '2025-09-03 12:56:44.973'::timestamp without time zone))
                        ->  Index Scan using lzl_202507_task_no_idx on lzl_202507 cc_7  (cost=0.43..633.45 rows=1973 width=56)
                              Index Cond: ((task_no)::text = (t.task_no)::text)
                              Filter: ((created_date > '2025-01-07 09:00:00'::timestamp without time zone) AND (created_date < '2025-09-03 12:56:44.973'::timestamp without time zone))
                        ->  Index Scan using lzl_202508_task_no_idx on lzl_202508 cc_8  (cost=0.43..619.43 rows=1720 width=56)
                              Index Cond: ((task_no)::text = (t.task_no)::text)
                              Filter: ((created_date > '2025-01-07 09:00:00'::timestamp without time zone) AND (created_date < '2025-09-03 12:56:44.973'::timestamp without time zone))
                        ->  Index Scan using lzl_202509_task_no_idx on lzl_202509 cc_9  (cost=0.42..893.03 rows=1521 width=56)
                              Index Cond: ((task_no)::text = (t.task_no)::text)
                              Filter: ((created_date > '2025-01-07 09:00:00'::timestamp without time zone) AND (created_date < '2025-09-03 12:56:44.973'::timestamp without time zone))

```

created_date时间范围，找1年的数据，前晚变更加的索引是created_date。

加索引后的执行计划：

```sql
             ->  Hash Join  (cost=63.37..23740.82 rows=191 width=33)
                    Hash Cond: ((cc.task_no)::text = (t.task_no)::text)
                    ->  Append  (cost=0.00..23376.98 rows=114435 width=58)
                          Subplans Removed: 28
                          ->  Index Scan using idx_lzltab_202501_created_date on lzltab_202501 cc_1  (cost=0.43..1450.59 rows=8958 width=66)
                                Index Cond: ((created_date > $1) AND (created_date < $2))
                          ->  Index Scan using idx_lzltab_202502_created_date on lzltab_202502 cc_2  (cost=0.43..1822.73 rows=7405 width=66)
                                Index Cond: ((created_date > $1) AND (created_date < $2))
                          ->  Index Scan using idx_lzltab_202503_created_date on lzltab_202503 cc_3  (cost=0.43..1430.03 rows=7917 width=57)
                                Index Cond: ((created_date > $1) AND (created_date < $2))
                          ->  Index Scan using idx_lzltab_202504_created_date on lzltab_202504 cc_4  (cost=0.43..2412.44 rows=11041 width=56)
                                Index Cond: ((created_date > $1) AND (created_date < $2))
                          ->  Index Scan using idx_lzltab_202505_created_date on lzltab_202505 cc_5  (cost=0.43..2260.73 rows=13381 width=56)
                                Index Cond: ((created_date > $1) AND (created_date < $2))
                          ->  Index Scan using idx_lzltab_202506_created_date on lzltab_202506 cc_6  (cost=0.43..3930.10 rows=17832 width=56)
                                Index Cond: ((created_date > $1) AND (created_date < $2))
                          ->  Index Scan using idx_lzltab_202507_created_date on lzltab_202507 cc_7  (cost=0.43..3878.77 rows=21786 width=56)
                                Index Cond: ((created_date > $1) AND (created_date < $2))
                          ->  Index Scan using idx_lzltab_202508_created_date on lzltab_202508 cc_8  (cost=0.43..4736.72 rows=22033 width=56)
                                Index Cond: ((created_date > $1) AND (created_date < $2))
                          ->  Index Scan using idx_lzltab_202509_created_date on lzltab_202509 cc_9  (cost=0.42..627.09 rows=1893 width=56)
                                Index Cond: ((created_date > $1) AND (created_date < $2))
                    ->  Hash  (cost=63.03..63.03 rows=27 width=24)
                          ->  Bitmap Heap Scan on ai_outbound_call_task t  (cost=2.99..63.03 rows=27 width=24)
                                Recheck Cond: ((created_by)::text = ($3)::text)
                                ->  Bitmap Index Scan on idx_ai_call_task_c  (cost=0.00..2.99 rows=27 width=0)
                                      Index Cond: ((created_by)::text = ($3)::text)
```

新的执行计划从task_no索引变成走created_date索引，nl变成hash join。cost从2259694减少到23740，减少100倍。但是，实际执行时间增加了10倍左右。



# 问题定位

从3个问题来带入分析定位：

1. 为什么优化人员建议了created_date索引？
2. 为什么走到了新的索引？
3. 为什么新的执行计划执行时间很长但预估rows很小？



## 为什么优化人员建议了created_date索引？

如果直接把pg日志中参数回填到sql文本，执行计划其实是好的那一个，也就是跑3s的走task_no索引的plan。优化人员也这么跑了，发现还可以。但是生产环境却不是这个执行计划。

即使强制不让走task_no索引，优化器选择全表扫描都不去走create date索引：

```sql
                     Hash Cond: (((cc.task_no)::text || ''::text) = (t.task_no)::text)
                     ->  Append  (cost=0.00..2794425.58 rows=22238757 width=57)
                           ->  Seq Scan on lzltab_202501 cc_1  (cost=0.00..193060.05 rows=1585238 width=66)
                                 Filter: ((created_date > '2025-01-08 11:00:00'::timestamp without time zone) AND (created_date < '2025-09-04 08:31:43'::timestamp without time zone))
                           ->  Seq Scan on lzltab_202502 cc_2  (cost=0.00..178567.54 rows=1480969 width=66)
                                 Filter: ((created_date > '2025-01-08 11:00:00'::timestamp without time zone) AND (created_date < '2025-09-04 08:31:43'::timestamp without time zone))
                           ->  Seq Scan on lzltab_202503 cc_3  (cost=0.00..191073.34 rows=1583356 width=57)
```

这就很奇怪，自己怎么跑都走不到created_date烂索引，生产环境又是怎么走到created_date的？

联想到有绑定变量，有可能会是generic plan。

generic plan的特性：

1. 当`force_custom_plan =auto`时，对比generic plan和avg(前五次硬解析的执行计划的代价)，如果generic plan更小就用generic plan，后续不再进行硬解析；否则每次都硬解析（见源码`choose_custom_plan`）
2. generic plan长什么样子跟绑定变量本身是多少没有关系



使用绑定变量传值很容易复现：

```sql
PREPARE sql1(timestamp without time zone,timestamp without time zone,text) AS
SELECT COUNT(*)
xxxxxxx...;

=# EXECUTE sql1('2025-01-08 11:00:00','2025-09-04 08:31:43','LIUZHILONG62'); 
 count 
-------
 12016
(1 row)

Time: 367.220 ms
=# EXECUTE sql1('2025-01-08 11:00:00','2025-09-04 08:31:43','LIUZHILONG62'); 
 count 
-------
 12016
(1 row)

Time: 254.386 ms
=# EXECUTE sql1('2025-01-08 11:00:00','2025-09-04 08:31:43','LIUZHILONG62'); 
 count 
-------
 12016
(1 row)

Time: 235.343 ms
=# EXECUTE sql1('2025-01-08 11:00:00','2025-09-04 08:31:43','LIUZHILONG62'); 
 count 
-------
 12016
(1 row)

Time: 234.110 ms
=# EXECUTE sql1('2025-01-08 11:00:00','2025-09-04 08:31:43','LIUZHILONG62'); 
 count 
-------
 12016
(1 row)

Time: 233.570 ms
=# EXECUTE sql1('2025-01-08 11:00:00','2025-09-04 08:31:43','LIUZHILONG62'); 
 count 
-------
 12016
(1 row)

Time: 70678.344 ms (01:10.678)  --第6次执行时间明显变长


=# select * from pg_prepared_statements\gx  --pg14支持pg_prepared_statements
generic_plans   | 1
custom_plans    | 5
```

前5次硬解析用custom_plans跑出来都很快，第6次用到generic_plan，走的created_date，即为生产的故障计划，非常慢。

所以优化建议里用created_date是有些问题，但是替换变量为具体值后explain出来的执行计划是对的，而生产环境业务用绑定变量跑，跑到了generic plan，故障就此出现。



## 为什么新的执行计划执行时间很长但预估rows很小？

故障执行计划有个问题，预估的cost太小，rows太少

```sql
                          ->  Index Scan using idx_lzltab_202501_created_date on lzltab_202501 cc_1  (cost=0.43..1450.59 rows=8958 width=66)
                                Index Cond: ((created_date > $1) AND (created_date < $2))
```

这从业务逻辑上看这有点不正常，因为create_date条件已经跨了多个分区，created_date是分区键，那么where created>=xx <=yy一定是连续的，在子分区上的选择率应该始终为1，rows应是子分区行数，应该是几百万而不是几千。

刚开始以为是统计信息的问题，但统计信息是比较准确的，202501的历史分区数据没有什么变动。

既然是generic plan需要从generic plan预估代价入手翻源码。cost要难一点，看rows的预估逻辑要相对简单些，也容易定位。

```c
static double
calc_rangesel(TypeCacheEntry *typcache, VariableStatData *vardata,
			  const RangeType *constval, Oid operator)
{
...
		else
		{
			/* with any other operator, empty Op non-empty matches nothing */
			selec = (1.0 - empty_frac) * hist_selec;
		}
	}

	/* all range operators are strict */
	selec *= (1.0 - null_frac);
```

`range_select=(1-null率)*直方图选择率`，range直方图选择率看range命中的直方图对应的选择率再加个命中的MCV即可。不过这个案例不用算这些。

因为generic plan不看直方图：

```c
/*
 * rangesel -- restriction selectivity for range operators
 */
Datum
rangesel(PG_FUNCTION_ARGS)
{
...
	/*
	 * If we got a valid constant on one side of the operator, proceed to
	 * estimate using statistics. Otherwise punt and return a default constant
	 * estimate.  Note that calc_rangesel need not handle
	 * OID_RANGE_ELEM_CONTAINED_OP.
	 */
	if (constrange)
		selec = calc_rangesel(typcache, &vardata, constrange, operator);
	else
		selec = default_range_selectivity(operator);
...
}
```

`calc_rangesel`就是上面那个有常量值传入的选择率计算，else就是默认选择率函数`default_range_selectivity`，不传入常量

```c
/*
 * Returns a default selectivity estimate for given operator, when we don't
 * have statistics or cannot use them for some reason.
 */
static double
default_range_selectivity(Oid operator)
{
	switch (operator)
	{
        ...
		case OID_RANGE_CONTAINS_ELEM_OP:
		case OID_RANGE_ELEM_CONTAINED_OP:

			/*
			 * "range @> elem" is more or less identical to a scalar
			 * inequality "A >= b AND A <= c".
			 */
			return DEFAULT_RANGE_INEQ_SEL;
	}
}
```

range的默认选择率define：

```c
/* default selectivity estimate for range inequalities "A > b AND A < c" */
#define DEFAULT_RANGE_INEQ_SEL	0.005
```

返回生产环境计算rows

```sql
 select reltuples::bigint*0.005 from pg_class where relname='lzltab_202501'\gx
-[ RECORD 1 ]------
?column? | 8958.350
```

这跟实际的rows估算值8958相等：

```sql
idx_lzltab_202501_created_date on lzltab_202501 cc_1  (cost=0.43..1450.59 rows=8958 width=66)
```

所以新的执行计划估算不准确是因为generic plan使用了默认的选择率。





# 问题总结

## 为什么要有generic plan以及软解析的问题

把generic plan看成是DEFAULT estimate plan还更容易理解一点

为什么generic plan好像总是有问题？

理出以下思维链：

generic plan是为了减少硬解析，即使用软解析
如果不是每次执行都是硬解析，那么可以不传入具体参数直接使用执行计划
如果不传参直接用执行计划，那么需要提前生成执行计划

提前生成执行计划的方式：

- 不传参执行计划（generic plan）
- 用前几次传参生成的执行计划（pg没有这种东西）



如果使用generic plan，有可能是不准确的，比如

- 1.数据倾斜（如某个mcv极高，如where a=1，但a=1非常多），这极度依赖传入参数本身是啥，但generic plan不传参，所以执行计划不可能准确
- 2.数据均衡但无法准确计算选择率（如where a>$1 and a<$2），如果不知道range没人能计算出选择率，但generic plan不传参，所以执行计划不可能准确



如果前几次传参执行计划（pg没有这种东西），有可能也是不准确的，比如

- 数据倾斜，前几次传参没有普遍性，前几次传参极大的影响了后续固定的执行计划是什么





## generic plan预估不准的问题分类

因为要5次对比，所以generic plan的问题可以分为2种：

1. 前5次SQL的执行没有普遍性。跟前5次执行计划相关性大，依赖数据倾斜和前5次参数是否具有普遍性。
2. generic plan本身有问题。generic plan因为数据倾斜或数据均衡但无法准确计算选择率，导致generic plan本身执行效率低下



## 优化方案



从这个案例看下来，generic plan问题在分区表上可能会出现，分区键是连续的，扫描所有分区建选择率应该为1，但generic plan为0.05，很可能导致走“全索引”扫描这种场景。

所以在优化的时候需要考虑更多：

- 不要建太多索引迷惑优化器
- 排除generic plan的干扰。用`EXECUTE`真实跑6次
- 会话级别`set plan_cache_mode='force_generic_plan'`; or  `set plan_cache_mode='force_custom_plan';`对比执行计划；或者在pg16+用explain (GENERIC_PLAN)对比执行计划



语法参考：

```sql
--prepare/excute
PREPARE sql1(text) AS
SELECT COUNT(*) FROM LZL where a=$1;

EXECUTE sql1('zzz');  --跑6次再说
EXPLAIN EXECUTE sql1('zzz');

select * from pg_prepared_statements --查看prepare语句信息，只能看当前会话

--对比执行计划，设置会话参数后执行EXPLAIN EXECUTE
set plan_cache_mode='force_generic_plan'
set plan_cache_mode='force_custom_plan'
--直接查看genetic plan,16+
explain (GENERIC_PLAN) xx  
```