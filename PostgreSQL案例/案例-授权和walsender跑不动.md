


# 现象

walsender LSN未推进，堆栈显示堵在pathman的`invalidate_psin_entries_using_relid`，relid改变，walsender cpu打满。

```sql
pstack 121327
#0  hash_seq_search (status=status@entry=0x7fffaadf8330) at dynahash.c:1441
#1  0x00002ba3b40ec728 in invalidate_psin_entries_using_relid (relid=relid@entry=42319501) at src/relation_info.c:251
#2  0x00002ba3b40ecb3d in forget_status_of_relation (relid=relid@entry=42319501) at src/relation_info.c:232
#3  0x00002ba3b40fcc96 in pathman_relcache_hook (arg=<optimized out>, relid=42319501) at src/hooks.c:934
#4  0x000000000087168a in LocalExecuteInvalidationMessage (msg=0x3a391c8) at inval.c:595
#5  0x000000000071d50e in ReorderBufferExecuteInvalidations (rb=0x1b63ff8, txn=0x1be5f58, txn=0x1be5f58) at reorderbuffer.c:2238
#6  ReorderBufferCommit (rb=0x1b63ff8, xid=xid@entry=4285897514, commit_lsn=405674661986920, end_lsn=<optimized out>, commit_time=commit_time@entry=799377897828299, origin_id=origin_id@entry=0, origin_lsn=origin_lsn@entry=0) at reorderbuffer.c:1819
#7  0x0000000000712d18 in DecodeCommit (xid=4285897514, parsed=0x7fffaadf8630, buf=0x7fffaadf87f0, ctx=0x1a359e8) at decode.c:637
#8  DecodeXactOp (ctx=0x1a359e8, buf=buf@entry=0x7fffaadf87f0) at decode.c:245
#9  0x00000000007130b2 in LogicalDecodingProcessRecord (ctx=0x1a359e8, record=0x1a35c80) at decode.c:114
#10 0x0000000000733662 in XLogSendLogical () at walsender.c:2885
#11 0x0000000000735942 in WalSndLoop (send_data=send_data@entry=0x733620 <XLogSendLogical>) at walsender.c:2287
#12 0x0000000000736692 in StartLogicalReplication (cmd=0x1846c68) at walsender.c:1213
#13 exec_replication_command (cmd_string=cmd_string@entry=0x181a288 "START_REPLICATION SLOT \"lzl_logical_rep\" LOGICAL 170F5/7C3EAE78 (\"proto_version\" '1', \"publication_names\" 'lzl_logical_rep')") at walsender.c:1640
#14 0x0000000000774e91 in PostgresMain (argc=<optimized out>, argv=argv@entry=0x1866478, dbname=0x18662b8 "lzldb", username=<optimized out>) at postgres.c:4325
#15 0x0000000000485989 in BackendRun (port=<optimized out>, port=<optimized out>) at postmaster.c:4526
#16 BackendStartup (port=0x18635b0) at postmaster.c:4210
#17 ServerLoop () at postmaster.c:1739
#18 0x0000000000702f08 in PostmasterMain (argc=argc@entry=3, argv=argv@entry=0x1814da0) at postmaster.c:1412
#19 0x000000000048660a in main (argc=3, argv=0x1814da0) at main.c:210

# 第二次执行，堆栈一致，relid改变
pstack 121327
#0  hash_seq_search (status=status@entry=0x7fffaadf8330) at dynahash.c:1441
#1  0x00002ba3b40ec728 in invalidate_psin_entries_using_relid (relid=relid@entry=26560221) at src/relation_info.c:251
#2  0x00002ba3b40ecb3d in forget_status_of_relation (relid=relid@entry=26560221) at src/relation_info.c:232
#3  0x00002ba3b40fcc96 in pathman_relcache_hook (arg=<optimized out>, relid=26560221) at src/hooks.c:934
#4  0x000000000087168a in LocalExecuteInvalidationMessage (msg=0x39f1f68) at inval.c:595
...
```

# 分析

relid改变说明walsender还在跑还没有死。LSN没有改变，可以分析LSN的位置，看看这个事务在做什么。

当时的信息还在的话，可以通过slot视图查restart LSN反查wal的位置。如果当时的信息不在，可以通过stack中的LSN来找到wal日志是哪个。

pg_waldump查看wal日志信息，过滤xid

```shell
rmgr: Heap        len (rec/tot):    961/   961, tx: 4285897514, lsn: 170F5/7DFE3470, prev 170F5/7DFE3430, desc: UPDATE+INIT off 2 xmax 4285897514 flags 0x00 ; new off 1 xmax 0, blkref #0: rel 1663/17662/1259 blk 8443, blkref #1: rel 1663/17662/1259 blk 7327

...
rmgr: Transaction len (rec/tot): 1778325/1778325, tx: 4285897514, lsn: 170F5/7E1F4268, prev 170F5/7E1F4220, desc: COMMIT 2025-05-01 09:24:57.828299 CST; inval msgs: catcache 22 catcache 22 catcache 22 catcache 22 catcache 50 catcache 49 catcache 50 catcache 49 catcache 50 catcache 49 catcache 50 catcache 49 catcache 50 catcache 49 catcache 50 
...
 relcache 48813261 relcache 48813255 relcache 51030741 relcache 48813252 relcache 50737247 relcache 48813246 relcache 48813243 relcache 48813237 relcache 50737241 relcache 48813234 relcache 48813224 relcache 49379811 relcache 48813216 relcache 48813210 relcache 45452775
```

rel 1663/17662/1259的record有18万条，最后一条record是inval msgs，catcache有7万，relcache 3万。

rel 1663/17662/1259是pg_class，通过xmin去查pg_class可以找到对应的表和xid提交时间：

```sql
select xmin,xmax,pg_xact_commit_timestamp(xmin),relname from pg_class where xmin='4285897514'::xid order by relname desc ;
    xmin    | xmax |   pg_xact_commit_timestamp    |                   relname                   
------------+------+-------------------------------+---------------------------------------------
 4285897514 |    0 | 2025-05-01 09:24:57.828299+08 | v$session
 4285897514 |    0 | 2025-05-01 09:24:57.828299+08 | tmp_20230801_id_seq
 4285897514 |    0 | 2025-05-01 09:24:57.828299+08 | tmp_20230801
 4285897514 |    0 | 2025-05-01 09:24:57.828299+08 | test_param
 4285897514 |    0 | 2025-05-01 09:24:57.828299+08 | test_20240105
 ...
 
 select count(*) from pg_class where xmin='4285897514'::xid ;
 count 
-------
 18523

 select count(*) from pg_class ;
 count  
--------
 139138
```

通过时间去查pglog日志：

```shell
2025-05-01 09:24:59.837 CST,"postgres","lzldb",61418,"[local]",6812cd65.efea,3,"DO",2025-05-01 09:24:53 CST,549/0,0,LOG,00000,"duration: 6036.275 ms  statement:     
...
                    EXECUTE 'GRANT SELECT ON ALL TABLES IN SCHEMA public TO r_lzldbdata_qry';
...
                END;
                $$",,,,,,,,,"psql","client backend"
```

基本可以定位是这个授权引起，因为授权会更新pgclass中的relacl，至少更新了1.8w个relation的权限，pgclass更新会触发invalidation消息，大量的invalidation消息在walsender进程中处理缓慢。



# 复现

```sql
--建逻辑复制链路，随便来个
select pg_create_logical_replication_slot('logical_test','test_decoding');
pg_recvlogical -h 127.0.0.1 -p 7997 -d lzldb -U repuser --slot=logical_test --start -f recv.sql &

--建多张表
DO $$
BEGIN
    FOR i IN 1..20000 LOOP
        EXECUTE format(
            'CREATE TABLE IF NOT EXISTS table_%s ( 
                col1 varchar(10)
            )', 
            lpad(i::text, 5, '0')  -- 生成5位数字编号的表名
        );
    END LOOP;
END $$;

--一次授权
grant select on all tables in schema public to r_lzldb_qry;
```

```sql
--完美复现
postgres@lzlhost:~/lzl/grant]$ pstack 172862
#0  hash_seq_search (status=status@entry=0x7ffd664be280) at dynahash.c:1444
#1  0x00002ad31235e728 in invalidate_psin_entries_using_relid (relid=relid@entry=1002857) at src/relation_info.c:251
#2  0x00002ad31235eb3d in forget_status_of_relation (relid=relid@entry=1002857) at src/relation_info.c:232
#3  0x00002ad31236ec96 in pathman_relcache_hook (arg=<optimized out>, relid=1002857) at src/hooks.c:934
#4  0x000000000087168a in LocalExecuteInvalidationMessage (msg=0x2ad3c3f61a88) at inval.c:595
#5  0x000000000071d50e in ReorderBufferExecuteInvalidations (rb=0x17e5698, txn=0x180d698, txn=0x180d698) at reorderbuffer.c:2238


[postgres@lzlhost:~/lzl/grant]$ pstack 172862
#0  0x0000000000891d0c in hash_seq_search (status=status@entry=0x7ffd664be280) at dynahash.c:1441
#1  0x00002ad31235e728 in invalidate_psin_entries_using_relid (relid=relid@entry=1011110) at src/relation_info.c:251
#2  0x00002ad31235eb3d in forget_status_of_relation (relid=relid@entry=1011110) at src/relation_info.c:232
#3  0x00002ad31236ec96 in pathman_relcache_hook (arg=<optimized out>, relid=1011110) at src/hooks.c:934


--relid在动
--cpu打满:
ps -eo pid,%cpu,%mem|grep 172862
172862 99.3  0.0

--运行2h左右追平
```



# without pathman加速walsender解析

因为库里面是没有用pathman分区表的，但有pathman extension，所以尝试绕过pathman hook以加速walsender解析过程。

```sql
drop extension pg_pathman;
grant update on all tables in schema public to r_lzldb_upd;

[postgres@lzlhost~/lzl/grant]$  pstack 133460
#0  hash_seq_search (status=status@entry=0x7ffe292d5c90) at dynahash.c:1418
#1  0x000000000087f228 in RelfilenodeMapInvalidateCallback (arg=<optimized out>, relid=1034036) at relfilenodemap.c:64
#2  0x000000000087168a in LocalExecuteInvalidationMessage (msg=0x2b9699795768) at inval.c:595
#3  0x000000000071d50e in ReorderBufferExecuteInvalidations (rb=0x195a358, txn=0x1a6ff38, txn=0x1a6ff38) at reorderbuffer.c:2238
#4  ReorderBufferCommit (rb=0x195a358, xid=xid@entry=328684387, commit_lsn=8016890875224, end_lsn=<optimized out>, commit_time=commit_time@entry=799851538975691, origin_id=origin_id@entry=0, origin_lsn=origin_lsn@entry=0) at reorderbuffer.c:1819

# 20s以内完成
```

没有注释`shared_preload_libraries`中的`pg_pathman`，同样有很好的效果，walsender从2小时缩短到20s。

这里感觉挺奇怪，没有注释`shared_preload_libraries`，hook应该还是要走的。分析源码发现，之所以有效果是因为hook的第一步就有pathman config表的判断，不会再走pathman中的invalidate逻辑了，所以很快可以执行完成：

```c
/*
 * Invalidate PartRelationInfo cache entry if needed.
 */
void
pathman_relcache_hook(Datum arg, Oid relid)
{
	Oid pathman_config_relid;

	/* See cook_partitioning_expression() */
	if (!pathman_hooks_enabled)
		return;

	if (!IsPathmanReady())
		return;

...

	/*
	 * Invalidation event for PATHMAN_CONFIG table (probably DROP EXTENSION).
	 * Digging catalogs here is expensive and probably illegal, so we take
	 * cached relid. It is possible that we don't know it atm (e.g. pathman
	 * was disabled). However, in this case caches must have been cleaned
	 * on disable, and there is no DROP-specific additional actions.
	 */
	pathman_config_relid = get_pathman_config_relid(true);
	if (relid == pathman_config_relid)
	{
		delay_pathman_shutdown();
	}

	/* Invalidation event for some user table */
	else if (relid >= FirstNormalObjectId)
	{
		/* Invalidate PartBoundInfo entry if needed */
		forget_bounds_of_rel(relid);

		/* Invalidate PartStatusInfo entry if needed */
		forget_status_of_relation(relid);

		/* Invalidate PartParentInfo entry if needed */
		forget_parent_of_partition(relid);
	}
}
```

`get_pathman_config_relid`获取了pathman_config表，`drop extension pg_pathman`把db中pathman_config表删掉了，所以源码就不会走forget那一堆逻辑。

也有其他办法可以加速walsender解析，比如将`pg_pathman.enable=off`，会走`if (!IsPathmanReady())`直接`return`。或者最直接的注释`shared_preload_libraries`中的`pg_pathman`并重启实例（这是实例级不是db级）。





# pg14的提升



pg14.0 release note:

> Allow logical decoding to more efficiently process cache invalidation messages (Dilip Kumar)  This allows logical decoding to work efficiently in presence of a large amount of DDL.

https://www.postgresql.org/docs/release/14.0/

patch:

<https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=d7eb52d71>



pg14`ReorderBufferAddInvalidations`的注释：

> We require to record it in form of the change so that we can execute only the required invalidations instead of executing all the invalidations on each CommandId increment. 



对比pg14和pg13，`ReorderBufferCommit`逻辑大改

13中处理事务的逻辑直接放在commit函数里`ReorderBufferCommit`

```c
				case REORDER_BUFFER_CHANGE_INTERNAL_COMMAND_ID:
					Assert(change->data.command_id != InvalidCommandId);

					if (command_id < change->data.command_id)
					{
						command_id = change->data.command_id;

						if (!snapshot_now->copied)
						{
							/* we don't use the global one anymore */
							snapshot_now = ReorderBufferCopySnap(rb, snapshot_now,
																 txn, command_id);
						}

						snapshot_now->curcid = command_id;

						TeardownHistoricSnapshot(false);
						SetupHistoricSnapshot(snapshot_now, txn->tuplecid_hash);

						/*
						 * Every time the CommandId is incremented, we could
						 * see new catalog contents, so execute all
						 * invalidations.
						 */
						ReorderBufferExecuteInvalidations(rb, txn);
					}
```

14中的主要逻辑是`ReorderBufferReplay`->`ReorderBufferProcessTXN`

在`ReorderBufferProcessTXN`中新增`case REORDER_BUFFER_CHANGE_INVALIDATION`判断以执行`ReorderBufferExecuteInvalidations`执行reorderbuffer中的invalidate

```c
				case REORDER_BUFFER_CHANGE_INVALIDATION:
					/* Execute the invalidation messages locally */
					ReorderBufferExecuteInvalidations(
													  change->data.inval.ninvalidations,
													  change->data.inval.invalidations);
					break;
```

`ReorderBufferExecuteInvalidations`之后的逻辑都差不多，没有多大变动。13对比14的`ReorderBufferCommit`主要差异：

- `ReorderBufferCommit`不再是主要process txn的函数，调用栈更深
- 新增`case REORDER_BUFFER_CHANGE_INVALIDATION`判断，与`REORDER_BUFFER_CHANGE_INTERNAL_COMMAND_ID`标识区分，单独处理invalidation
- 移除以每个command_id处理invalidation的逻辑





# 根因和解决办法

walsender卡住的根因是有批量授权任务，导致pg_class多个元组更新，触发大量invalidation消息。`GRANT privs ON ALL TABLES IN SCHEMA public TO role1` 这样的授权语句在pg中是1个事务多个command处理的，而pg13 逻辑复制处理invalidation message时，会通过每个command去处理一次reorder buffer invalidate，并处理各个hook的inval hash表。在这个场景中，pathman hook在处理inval hash表较慢，从而导致复制延迟。

pathman处理较慢的触发条件（且）：

- pg13-
- 批量授权
- 安装了pathman extension（无论是否使用）
- 逻辑复制链路



即使将pathman干掉，也会在RelfilenodeMapInvalidateCallback等函数上花较多cpu。pg13目前测下来，有pathman对比没有pathman，处理时间是小时级和分钟级的差距。

其他未测试但社区提及的场景（且）：

- pg13-
- 批量DDL/TRUNCATE/DCL/DROP PUBLICATION
- 逻辑复制链路



短期解决办法：如果没有用到pathman表的，可以drop extension或者不加载pathman so；重启链路

长期解决办法：升级到pg14+（已测试非常快无延迟）


## 

# ref

<https://www.postgresql.org/message-id/flat/17716-1fe42e7b44fc2f25%40postgresql.org>

<https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=d7eb52d71>



