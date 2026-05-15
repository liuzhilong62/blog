---
layout: post
title: "从collation mismatch异常到其原理"
date: 2025-12-13
categories: [PostgreSQL内功修炼]
description: "从collation mismatch异常入手，深入解析PostgreSQL排序规则版本管理与操作系统libc依赖原理。"
---

# 问题现象

物理迁移信创后pg log偶有报错，版本是pg15：

```shell
WARNING:  01000: collation "zh_CN.utf8" has version mismatch
DETAIL:  The collation in the database was created using version 2.17, but the operating system provides version 2.28.
HINT:  Rebuild all objects affected by this collation and run ALTER COLLATION pg_catalog."zh_CN.utf8" REFRESH VERSION, or build RaseSQL with the right library version.
LOCATION:  pg_newlocale_from_collation, pg_locale.c:1660
```

前景：物理切换的时候做了失效索引重建和refresh database collation version。

虽然物理迁移后libc版本升了，但是做了索引重建的，索引现在是有效的，而且db中的collation  version已经与OS libc一致。

所以，

为什么会报错？

报错哪里触发的？

报错有什么影响？

怎么解决？

# 问题分析

## 为什么会报错？

数据库内部的collation主要看3块：db、列、索引。前两个都是默认collation，索引的collation才是真collation。

先检查database的collation，db的都是en_US.UTF8，且已经refresh database collation过了，所以“collation "zh_CN.utf8" has version mismatch”这个报错不应该在db层抛出。

然后再查看字段没有特殊指定的默认collation：

```sql
select attrelid,attname,attcollation from pg_attribute where attcollation not in (0,100,950,951);
 attrelid | attname | attcollation 
----------+---------+--------------
(0 rows)
```

0表示没有collation，default oid=100,C oid=950,POSIX oid=951；"zh_CN.utf8"肯定不会是这四个。

最后再查看索引没有特殊指定的collation

```sql
select * from (select indexrelid ,unnest(indcollation) coll from pg_index) i where coll not in (0,100,950,951);
 indexrelid | coll 
------------+------
(0 rows)
```

排除db、字段、索引，那么只能是一种情况：业务层指定排序规则

```sql
select col1 from (values ('a'), ('A'), ('啊'), ('阿')) AS l(col1) order by col1 collate "zh_CN.utf8";
WARNING:  01000: collation "zh_CN.utf8" has version mismatch
DETAIL:  The collation in the database was created using version 2.17, but the operating system provides version 2.28.
HINT:  Rebuild all objects affected by this collation and run ALTER COLLATION pg_catalog."zh_CN.utf8" REFRESH VERSION, or build RaseSQL with the right library version.
LOCATION:  pg_newlocale_from_collation, pg_locale.c:1660
 col1 
------
 阿
 啊
 a
 A
```

这个zh_CN.utf8的version就跟实际的不一致：

```sql
  select collname,collversion,pg_collation_actual_version(oid) from pg_collation where collname ='zh_CN.utf8';
  collname  | collversion | pg_collation_actual_version 
------------+-------------+-----------------------------
 zh_CN.utf8 | 2.17        | 2.28
```

不仅zh_CN.utf8不一样，所有都不一样（除了几个没有version说法的coll）

所以很有可能是业务自己指定了一个排序规则"zh_CN.utf8"，但是库中的coll version与OS不一致，才抛出了报错。



## 源码理解

通过报错信息是很好定位到源码位置的，主要关注2个函数：`pg_newlocale_from_collation`和`CheckMyDatabase`。



### `pg_newlocale_from_collation`缓存和检查`pg_collation`

`pg_newlocale_from_collation`是pg10才有

```c
/*
 * Create a locale_t from a collation OID.  Results are cached for the
 * lifetime of the backend.  Thus, do not free the result with freelocale().
 *
 * As a special optimization, the default/database collation returns 0.
 * Callers should then revert to the non-locale_t-enabled code path.
 * In fact, they shouldn't call this function at all when they are dealing
 * with the default locale.  That can save quite a bit in hotspots.
 * Also, callers should avoid calling this before going down a C/POSIX
 * fastpath, because such a fastpath should work even on platforms without
 * locale_t support in the C library.
 *
 * For simplicity, we always generate COLLATE + CTYPE even though we
 * might only need one of them.  Since this is called only once per session,
 * it shouldn't cost much.
 */
/* locale_t指非ICU。该函数是为backend cache一个locale_t类型的collation OID
* the default/database collation returns 0。其中default表示使用db的collation
*/
pg_locale_t
pg_newlocale_from_collation(Oid collid)  //注意传入的是collation的oid，不是拿所有pg_collation
{
...
	/* Return 0 for "default" collation, just in case caller forgets */
	if (collid == DEFAULT_COLLATION_OID)  //有三个特殊的coll：
		return (pg_locale_t) 0;       //default oid=100,C oid=950,POSIX oid=951

...
	if (cache_entry->locale == 0)
	{
...
		collversion = SysCacheGetAttr(COLLOID, tp, Anum_pg_collation_collversion,
									  &isnull);  //从pg_collation中拿数据字典中的version
		if (!isnull)
		{
...
			actual_versionstr = get_collation_actual_version(collform->collprovider, collcollate); //通过get_collation_actual_version获得实际的version
...
			collversionstr = TextDatumGetCString(collversion);

			if (strcmp(actual_versionstr, collversionstr) != 0)  //对比数据字典中的version和实际的version，不同就抛出报错
				ereport(WARNING,
						(errmsg("collation \"%s\" has version mismatch",
								NameStr(collform->collname)),
						 errdetail("The collation in the database was created using version %s, "
								   "but the operating system provides version %s.",
								   collversionstr, actual_versionstr),
						 errhint("Rebuild all objects affected by this collation and run "
								 "ALTER COLLATION %s REFRESH VERSION, "
								 "or build PostgreSQL with the right library version.",
								 quote_qualified_identifier(get_namespace_name(collform->collnamespace),
															NameStr(collform->collname)))));
		}
...
	return cache_entry->locale;
}
```

主要是通过coll oid检查这个coll在pg_collation数据字典的version和实际的version是否一致，如果不一致就抛出报错。



### `CheckMyDatabase`缓存和检查`pg_database`

`CheckMyDatabase`已经存在很久了，它要做很多db方面的检查。不过在pg15增加了检查db version的逻辑

```c
/*
 * CheckMyDatabase -- fetch information from the pg_database entry for our DB
 */
static void
CheckMyDatabase(const char *name, bool am_superuser, bool override_allow_connections)
{
...
	/* Fetch our pg_database row normally, via syscache */
	tup = SearchSysCache1(DATABASEOID, ObjectIdGetDatum(MyDatabaseId));
...
	default_locale.provider = dbform->datlocprovider; //default就是db的

	/*
	 * Default locale is currently always deterministic.  Nondeterministic
	 * locales currently don't support pattern matching, which would break a
	 * lot of things if applied globally.
	 */
	default_locale.deterministic = true; //字节序敏感的

	/*
	 * Check collation version.  See similar code in
	 * pg_newlocale_from_collation().  Note that here we warn instead of error
	 * in any case, so that we don't prevent connecting.
	 */
	datum = SysCacheGetAttr(DATABASEOID, tup, Anum_pg_database_datcollversion,
							&isnull); //从pg_database中拿datcollversion
	if (!isnull)
	{
		char	   *actual_versionstr;
		char	   *collversionstr;

		collversionstr = TextDatumGetCString(datum);

		actual_versionstr = get_collation_actual_version(dbform->datlocprovider, dbform->datlocprovider == COLLPROVIDER_ICU ? iculocale : collate);  //通过get_collation_actual_version获得实际的version
...
		else if (strcmp(actual_versionstr, collversionstr) != 0) //对比db datcollversion和实际的version，不相等则抛出warning
			ereport(WARNING,
					(errmsg("database \"%s\" has a collation version mismatch",
							name),
					 errdetail("The database was created using collation version %s, "
							   "but the operating system provides version %s.",
							   collversionstr, actual_versionstr),
					 errhint("Rebuild all objects in this database that use the default collation and run "
							 "ALTER DATABASE %s REFRESH COLLATION VERSION, "
							 "or build PostgreSQL with the right library version.",
							 quote_identifier(name))));
	}
...
}
```

`CheckMyDatabase`函数会对比数据字典pg_database中的datcollversion和实际的version。



### 函数差异

- 在pg14及之前，对比collation的逻辑只有1个:会话首次缓存对应collation时，调用```pg_newlocale_from_collation```访问**pg_collation数据字典中的对应的那个collation的version**，对比真实的version
- 在PG15及以后，因为在pg_database表中新增了datcollversion字段，所以新增了一个检查db collation version的逻辑：会话首次访问pg_database中的db时，调用`CheckMyDatabase`检查**pg_database中对应的那个db的datcollversion**，对比真实的version



## 为什么仅refresh database后报错不多

refresh database collation version后，不会触发pg_database的coll version不一致的warning，但是也不能排除pg_collation的coll version不一致的情况。为什么仅refresh database后，报错就少量这么多，难道pg_collation的coll version就不会被加载了?

```sql
select c.coll,count(*) from (select  unnest(indcollation) coll from pg_index ) c group by c.coll;
 coll | count 
------+-------
  950 |    37  --C
    0 |  2841  --不存在collation
  100 |   723  --default
```

真实环境中，用的最多的还是default，一般也没人去指定collation，没有指定就是default，default就是db的默认collation。

这里要回来再次关注`pg_newlocale_from_collation`函数。函数刚开头是这样的：

```c
pg_locale_t
pg_newlocale_from_collation(Oid collid)
{
	collation_cache_entry *cache_entry;

	/* Callers must pass a valid OID */
	Assert(OidIsValid(collid));

	/* Return 0 for "default" collation, just in case caller forgets */
	if (collid == DEFAULT_COLLATION_OID)
		return (pg_locale_t) 0;
...
```

当`collid==DEFAULT_COLLATION_OID`==100时，直接`return`，不再执行下面的真实version的判断，也就不会抛出warning。这个逻辑是合理的，因为db coll version已经在登入db时校验过了，如果有问题，那么肯定已经在session层抛出过一次warning了。

另外，即便是传入可能的值collid=37，对应的C也没有version的说法。

所以，refresh database后，绝大部分场景下，只要是用数据库内部的排序（非表达式排序和指定索引排序），就不会抛出报错了。



## 测试

这里仅测试是否有refresh warning，不测试索引corrupt或者库跑崩。

```shell
#查看libc版本
getconf GNU_LIBC_VERSION

源主机版本 glibc 2.17
目标主机   glibc 2.28
pg版本    pg15+
```

### 测试：刷db不刷新pg_collation，仅db coll version改变

```sql
select datname,datlocprovider,datcollate,datctype,datcollversion from pg_database 
  datname   | datlocprovider | datcollate  |  datctype   | datcollversion 
------------+----------------+-------------+-------------+----------------
 lzldb      | c              | en_US.UTF-8 | en_US.UTF-8 | 2.17
 
select collname,collprovider,collversion,pg_collation_actual_version(oid) from pg_collation where collname ~ 'en_US.utf8';
  collname  | collprovider | collversion | pg_collation_actual_version 
------------+--------------+-------------+-----------------------------
 en_US.utf8 | c            | 2.17        | 2.28

alter database lzldb refresh collation version;
NOTICE:  00000: changing version from 2.17 to 2.28
LOCATION:  AlterDatabaseRefreshColl, dbcommands.c:2399
ALTER DATABASE
```

再次查看pg_collation和pg_database

```sql
  collname  | collprovider | collversion | pg_collation_actual_version 
------------+--------------+-------------+-----------------------------
 en_US.utf8 | c            | 2.17        | 2.28

  datname   | datlocprovider | datcollate  |  datctype   | datcollversion 
------------+----------------+-------------+-------------+----------------
 lzldb      | c              | en_US.UTF-8 | en_US.UTF-8 | 2.28
```

跟官方文档的描述一致，refresh database collation version只是在刷db的默认collation，pg_collation本身是不会变的

### 测试：刷db不刷新pg_collation，指定表达式排序报warning

如开头分析，表达式排序会报warning，略

### 测试：刷db不刷新pg_collation，新建指定collation的索引报warning

测试1：建索引时指定collation

```sql
   collname  | collversion | pg_collation_actual_version 
------------+-------------+-----------------------------
 zh_CN.utf8 | 2.17        | 2.28
 
 > create index idx11 on tt(a collate "zh_CN.utf8");
WARNING:  01000: collation "zh_CN.utf8" has version mismatch
DETAIL:  The collation in the database was created using version 2.17, but the operating system provides version 2.28.
HINT:  Rebuild all objects affected by this collation and run ALTER COLLATION pg_catalog."zh_CN.utf8" REFRESH VERSION, or build PostgreSQL with the right library version.
LOCATION:  pg_newlocale_from_collation, pg_locale.c:1664
CREATE INDEX
```

测试2：建表时指定字段默认collation，建索引时不指定

```sql
\c lzldb   --重连一个session
You are now connected to database "lzldb" as user "postgres".
create table ttt(a varchar(10) collate "zh_CN.utf8");
CREATE TABLE

> create index idxttt on ttt(a);
WARNING:  01000: collation "zh_CN.utf8" has version mismatch
DETAIL:  The collation in the database was created using version 2.17, but the operating system provides version 2.28.
HINT:  Rebuild all objects affected by this collation and run ALTER COLLATION pg_catalog."zh_CN.utf8" REFRESH VERSION, or build PostgreSQL with the right library version.
LOCATION:  pg_newlocale_from_collation, pg_locale.c:1664
CREATE INDEX
Time: 7.904 ms
```

字段默认collation和建索引指定collation本质是一个东西，都是为了指定索引的collation。他们都可以报warning。

### 测试：刷db不刷新pg_collation，已建指定collation的索引不报warning

场景为原库已经有索引指定collation zh_CN.utf8，跟db的不一样，refresh db刷不到。但是到新库后vendor的coll version肯定是变了。

```sql
select collname,collprovider,collversion,pg_collation_actual_version(oid) from pg_collation where collname ~ 'zh_CN.utf8';
  collname  | collprovider | collversion | pg_collation_actual_version 
------------+--------------+-------------+-----------------------------
 zh_CN.utf8 | c            | 2.17        | 2.28
```

不用表达式排序的话，可以用到索引，但是不能使用索引排序

```sql
> set enable_seqscan =off;
SET
> EXPLAIN ANALYZE SELECT a  FROM tt  ORDER BY a LIMIT 1000;
                                                               QUERY PLAN                                                               
----------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=6667.80..6670.30 rows=1000 width=33) (actual time=44.928..45.145 rows=1000 loops=1)
   ->  Sort  (cost=6667.80..6892.81 rows=90004 width=33) (actual time=44.926..45.021 rows=1000 loops=1)
         Sort Key: a
         Sort Method: top-N heapsort  Memory: 127kB
         ->  Index Only Scan using idxtt on tt  (cost=0.42..1732.98 rows=90004 width=33) (actual time=0.029..15.434 rows=90004 loops=1)
               Heap Fetches: 4
```

已建指定collation的索引，使用到的时候不报warning



# 这个问题的总结

refresh database和refresh collation的warning是session级。在每个会话中，对于每个db或每个collation只报一次。

仅refresh database，很有可能不会再报warning，但是存在创建指定collation索引、运行指定表达式collation SQL时报warning的情况。

数据字典中的coll version只是为了在db层追踪collation provider version是否发生改变。试想一下没有数据字典中的coll version，那么db可能都无法返回warning告知“你的排序规则提供商升级了版本，你的数据排序可能有问题，你需要检查一下了”（当然不止是排序）。



# 这个问题的解决方案

corrupt的索引已经重建过，db也refresh过，只是没有refresh collation。数据字典中的coll version不一致，问题不算太大，只是提示而已，至于还有其他什么隐秘又奇怪的坑，参考more章节。

这个问题的解决方案：

第一步：检查是否还有依赖

```sql
SELECT pg_describe_object(refclassid, refobjid, refobjsubid) AS "Collation",
       pg_describe_object(classid, objid, objsubid) AS "Object"
  FROM pg_depend d JOIN pg_collation c
       ON refclassid = 'pg_collation'::regclass AND refobjid = c.oid
  WHERE c.collversion <> pg_collation_actual_version(c.oid)
  ORDER BY 1, 2;

```

如果有返回，最好重建依赖的对象；没有的话遵循第二步：

- 方案一：不动。warning不多也可以不动
- 方案二：仅refresh collation zh_CN.UTF8。来一个解决一个
- 方案三：所有collation都refresh一遍。哪怕业务有增量使用表达式或索引指定collation，也不会报warning



# more

## glibc升级相关的精华总结

locale是一个非常坑的领域，glibc升级导致的collation的问题也非常多，参考ref资料，汇总一些比较重要的东西：



pg_collation是从OS命令`locale -a`取的；provider基本都是glibc，所以要看glibc的version。

在pg_collation中"C"和"posix"的collprovider都是`c`，看着跟“C.UTF8”等一样，其实不是的。“C.UTF8”的provider是glibc，**有version，一般是unicode码点排序或unicode语义排序**；"C"和“POSIX”是等价的，是POSIX 标准定义的最基础的locale，由libc实现，不在`locale -a`中，**没有version，直接以字节序排序**。

collation问题的根源：数据库要的是数据库生命周期内locale定义永远不变，但是OS提供商特别是GNU C library每个小版本都会对locale做出改动，而且这是合法的。

GNU C library每个小版本都会对locale做出改动，现实中最容易出问题的版本是**glibc 2.28**，因为2.28升级了大版本**unicode 9.0.0**([has been updated to a new upstream version from ISO which is in sync with Unicode 9.0.0](https://sourceware.org/glibc/wiki/Release/2.28))。

**pg没有办法检测glibc升级带来的兼容性问题**。索引损坏检查不是all check，同时索引也只是一方面。物理复制或upgrade后，即使重建索引，也不能排除因为collation version的问题，某天库跑崩了的情况。

数据异常包括：主键重复，依赖排序的约束，范围分区表数据写入错误分区，mergejoin等排序操作等等

字符类型依赖collation，不依赖collation的数据类型：

- bytea
- tsvector gin indexes
- pg_trgm indexes
- numeric data types: int, bigint, numeric, float, ...
- custom data types like geometry (PostGIS)
- timestamp

ASCII排序比较常见但不符合人类理解，即不符合语义。符合语义的国际排序标准一般都是unicode标准。

基于unicode的排序规则也分了好2种：码点排序、UCA（Unicode Collation Algorithm）

UCA基于DUCET（Default Unicode Collation Element Table），DUCET表本身在不同版本间可能发生排序变化。举个栗子，en_US.UTF8是UCA排序，相当于语义排序，版本升级会改变排序规则。C.UTF8是码点排序，码点确认后不会改变，不会改变排序规则。

PG 17+提供了非常安全的locale提供方式：builtin，不再依赖OS提供的glibc、ICU等provider。启用命令例如

```sql
initdb --locale-provider=builtin --bultin-locale=C.UTF-8 dbname1

```

17仅支持C, C.UTF-8，C是字节序排序（约等于ASCII排序），C.UTF-8是unicode码点排序；18多一个PG_UNICODE_FAST ，也是unicode码点排序，跟C.UTF-8[稍有区别](https://www.postgresql.org/docs/18/locale.html#LOCALE-PROVIDERS)。

因为数据库必须保持排序稳定不变，业务自定义排序只有推给业务层实现，例如表达式排序就是语义明确的，而且不影响数据库本身对collation的选择。如果哪天pg也支持build-in en_US.utf8的话，再考虑build-in的语义排序。

信创迁移时，信创主机的glibc版本一般都比老的英特尔服务器glibc版本高，很可能跨了2.28这个版本。加上任务急、kpi推动、人力不足和大库，物理迁移是在所难免。所以信创物理迁移得关注glibc版本和collation导致的许多异常。



## 物理迁移后可以做什么

假设数据库是en_US.utf8，provider c，已经做了跨libc版本的物理迁移，应该做如下操作:

一、官方必修方案

1.至少要重建有问题的索引。安装amcheck插件，用bt_index_check函数 

```sql
SELECT bt_index_check('idx1'::regclass, true);

```

2.refresh database version，(pg15+)

```sql
ALTER DATABASE name REFRESH COLLATION VERSION

```

3.检查有没有其他[依赖的对象](https://www.postgresql.org/docs/18/sql-altercollation.html#SQL-ALTERCOLLATION-NOTES)，有的话得看情况处理了

```sql
SELECT pg_describe_object(refclassid, refobjid, refobjsubid) AS "Collation",
       pg_describe_object(classid, objid, objsubid) AS "Object"
  FROM pg_depend d JOIN pg_collation c
       ON refclassid = 'pg_collation'::regclass AND refobjid = c.oid
  WHERE c.collversion <> pg_collation_actual_version(c.oid)
  ORDER BY 1, 2;

```

处理完以后，再

4.refresh collation version，(pg10+)

```sql
ALTER COLLATION name REFRESH VERSION

```



二、非官方邪修方案

我这没有做出完整的方案，只是一点思路。

1.处理分区表写入错误分区的问题

分区键是int/bigint/float，跟collation没有关系，可以不用管了

分区键是时间分区，如果是timestamp不用管了，如果是varchar等等字符类型，就看情况了

分区键是字符类型，参考“a”和“-”的排序（pgconf Collation Challenges Sorting It Out）。但要注意以下几点

- 如果要查数据的话，不要从父表查，可能会崩或者查不出来
- 没有简单的检测方案

2.处理主键/唯一键冲突

3.处理fdw排序范围异常的问题

4.未知问题



# ref

https://wiki.postgresql.org/wiki/Locale_data_changes

https://wiki.postgresql.org/wiki/Collations

pgconf Collation Challenges Sorting It Out

PFCONF Collations from A to Z

http://www.unicode.org/reports/tr10/tr10-34.html

https://sourceware.org/glibc/wiki/Release/2.28

https://www.postgresql.org/docs/18/sql-altercollation.html

https://www.postgresql.org/docs/18/sql-alterdatabase.html

https://www.postgresql.org/docs/17/locale.html#LOCALE-PROVIDERS