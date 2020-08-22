
# PostgreSQL DBA  SQL 
- [PostgreSQL DBA  SQL](#postgresql-dba-sql)
  - [脚本信息](#脚本信息)
  - [会话管理](#会话管理)
  - [vacummm](#vacummm)
  - [性能问题分析](#性能问题分析)
    - [TOP SQL](#top-sql)
    - [索引缺失](#索引缺失)
  - [对象管理](#对象管理)
    - [对象查看](#对象查看)
  - [参考](#参考)

## 脚本信息
Auth: Hezhiyong    
日期：2020-8-22    
分类：postgres ,DBA,SQL    
用途: DBA 日常管理运维     

## 会话管理 
停掉或者kill掉卡住的会话   
a)优先在数据库操作查询活跃的后台会话：
```sql
--查看活动会话数
select p.datname,p.usename,p.application_name,p.client_addr,p.query_start,
       p.current_query,p.waiting,p.procpid 
from pg_stat_activity p ; 
SELECT client_addr ,client_port,wait_event,username,query_start,substr(query,0,40) as sql FROM pg_stat_activity;
--查看用户连接
SELECT client_addr ,client_port,wait_event,query_start,query FROM pg_stat_activity;
SELECT  usename,client_addr ,wait_event,count(*) FROM pg_stat_activity group by usename,client_addr ,wait_event ;
--查看ip 连接 情况
select usename,client_addr ,count(*) from pg_stat_activity group by usename,client_addr;
select datname,state_change,wait_event,substr(query,1,100) as  sql  from pg_stat_activity limit 30; 
select application_name,datname,substr(query,1,100) as  sql  from pg_stat_activity limit 30; 

--查询长事务 
select * from pg_stat_activity where state <> 'idle' and now() - xact_start > '7200 sec'::interval;

--lock
with recursive t_wait as
   (select a.locktype,a.database,a.relation,a.page,a.tuple,a.classid,a.objid,a.objsubid,a.pid,a.virtualtransaction,a.virtualxid ,a.transactionid from pg_locks a where not a.granted),
     t_run as
          (select
              a.mode,a.locktype,a.database,a.relation,a.page,a.tuple,a.classid,a.objid,a.objsubid,a.pid
              ,a.virtualtransaction,a.virtualxid,a,transactionid,b.query,b.xact_start,b.query_start
              ,b.usename,b.datname from pg_locks a,pg_stat_activity b
          where a.pid=b.pid and a.granted)
    ,w  as(select r.pid r_pid, w.pid w_pid
          from t_wait w,t_run r where
              r.locktype is not distinct from w.locktype and
              r.database is not distinct from w.database and
              r.relation is not distinct from w.relation and
              r.page is not distinct from w.page and
              r.tuple is not distinct from w.tuple and
              r.classid is not distinct from w.classid and
              r.objid is not distinct from w.objid and
              r.objsubid is not distinct from w.objsubid and
              r.transactionid is not distinct from w.transactionid and
              r.virtualxid is not distinct from w.virtualxid)
    ,c(waiter, holder, root_holder, path, deep) as(
          select w_pid, r_pid, r_pid, w_pid||'->'||r_pid, 1 from w
          union
          select w_pid, r_pid, c.holder, w_pid||'->'||c.path, c.deep+1 from w t, c where t.r_pid = c.waiter
      )
select t1.waiter, t1.holder, t1.root_holder, path, t1.deep from c t1 where
    not exists(select 1 from c t2 where t2.path ~ t1.path and t1.path<>t2.path )
Order by root_holder;
--查看被锁定的表
SELECT pg_class.relname AS table, pg_database.datname AS database, pid, mode, granted  FROM pg_locks, pg_class, pg_database
WHERE pg_locks.relation = pg_class.oid   AND pg_locks.database = pg_database.oid;

```

kill 会话
```
select pg_cancel_backend('procpid');  --取消session  
select pg_terminate_backend('procpid');         --结束session 
```     
> **说明**        
     pg_cancel_backend()操作后，session还在，事物回退;    
     pg_terminate_backend()操作后，session消失，事物回退。    
     如果在某些时候pg_terminate_backend()不能杀死session，那么可以在os层面，使用kill命令.    


b）在操作系统kill
```bash
ps –ef|grep postgresql ##第二个字段pid，找到需要kill的那个进程 
kill pid 
kill -9  pid   ##优先使用kill，kill -9的权限很高，可能引起故障 
```
## vacummm    

查看vocuum 进程
```sql
  select pid,datname,usename,query,xact_start 
   from pg_stat_activity where query like '%vacuum%' and query not like 'select%;
```
## 性能问题分析

### TOP SQL
```sql
--运行时间最久top 10
SELECT 	pd.datname,pss.query AS SQLQuery ,pss.rows AS TotalRowCount
	,(pss.total_time / 1000 / 60) AS TotalMinute 
	,((pss.total_time / 1000 / 60)/calls) as TotalAverageTime		
FROM pg_stat_statements AS pss
INNER JOIN pg_database AS pd
	ON pss.dbid=pd.oid
ORDER BY 4 DESC 
LIMIT 10;

--最耗IO SQL
--单次调用最耗IO SQL TOP 5
select userid::regrole, dbid, query from pg_stat_statements order by (blk_read_time+blk_write_time)/calls desc limit 5;  
--总最耗IO SQL TOP 5
select userid::regrole, dbid, query from pg_stat_statements order by (blk_read_time+blk_write_time) desc limit 20;  
--最耗时 SQL
--单次调用最耗时 SQL TOP 5
select userid::regrole, dbid, query from pg_stat_statements order by mean_time desc limit 5;  
--总最耗时 SQL TOP 5
select userid::regrole, dbid, query from pg_stat_statements order by total_time desc limit 5;  
--响应时间抖动最严重 SQL
select userid::regrole, dbid, query from pg_stat_statements order by stddev_time desc limit 5;  
--最耗共享内存 SQL
select userid::regrole, dbid, query from pg_stat_statements order by (shared_blks_hit+shared_blks_dirtied) desc limit 5;  
--最耗临时空间 SQL
select userid::regrole, dbid, query from pg_stat_statements order by temp_blks_written desc limit 5;  
select * from pg_stat_statements order by total_time desc limit 5;
```


查询读取Buffer次数最多的SQL，这些SQL可能由于所查询的数据没有索引，而导致了过多的Buffer读，也同时大量消耗了CPU
``` sql
select * from pg_stat_statements order by shared_blks_hit+shared_blks_read desc limit 5;
```

第二种方法是，直接通过pg_stat_activity视图，利用下面的查询，查看当前长时间执行，一直不结束的SQL。这些SQL对应造成CPU满，也有直接嫌疑。
``` sql
select datname, usename, client_addr, application_name, state, backend_start, xact_start, xact_stay, 
query_start, query_stay, replace(query, chr(10), ' ') as query from (select pgsa.datname as datname, 
pgsa.usename as usename, pgsa.client_addr client_addr, pgsa.application_name as application_name, 
pgsa.state as state, pgsa.backend_start as backend_start, pgsa.xact_start as xact_start, 
extract(epoch from (now() - pgsa.xact_start)) as xact_stay, pgsa.query_start as query_start, 
extract(epoch from (now() - pgsa.query_start)) as query_stay , pgsa.query as query 
from pg_stat_activity as pgsa where pgsa.state != 'idle' and pgsa.state != 'idle in transaction' and
 pgsa.state != 'idle in transaction (aborted)') idleconnections order by query_stay desc limit 5;

```
### 索引缺失

[外键是否丢失索引](https://dba.stackexchange.com/questions/121976/discover-missing-foreign-keys-and-or-indexes)
```sql
WITH source_data AS
(
    SELECT table_name AS source_table_name, column_name AS source_column_name
    FROM information_schema.columns
    WHERE table_schema = 'public'
    AND table_name LIKE 't_%'
    AND column_name LIKE '%\_id'
), target_data AS
(
    SELECT distinct table_name as target_table_name
    FROM information_schema.columns
    WHERE table_schema = 'public'
    AND table_name like 't_%'
), data_to_find AS
(
    SELECT *, CASE WHEN target_table_name IS NULL THEN 'No Match' ELSE 'Match' END AS match_found
    FROM source_data
    LEFT JOIN target_data ON ('t_' || LEFT(source_column_name, -3)) = target_table_name
), foreign_keys AS
(
    SELECT
        tc.constraint_name, tc.table_name, kcu.column_name, 
        ccu.table_name AS foreign_table_name,
        ccu.column_name AS foreign_column_name 
    FROM 
        information_schema.table_constraints AS tc 
        JOIN information_schema.key_column_usage AS kcu
          ON tc.constraint_name = kcu.constraint_name
        JOIN information_schema.constraint_column_usage AS ccu
          ON ccu.constraint_name = tc.constraint_name
    WHERE constraint_type = 'FOREIGN KEY'
)
SELECT CASE WHEN constraint_name IS NULL THEN 'No Foreign Key' ELSE 'Foreign Key Found' end AS foreign_key, * FROM data_to_find
LEFT JOIN foreign_keys ON 
(
    data_to_find.source_table_name = foreign_keys.table_name 
    AND
    data_to_find.source_column_name = foreign_keys.column_name
)
ORDER BY foreign_key, match_found, source_table_name, source_column_name;


```

[查找是否有全表扫描的sql](http://www.postgresonline.com/journal/archives/65-How-to-determine-which-tables-are-missing-indexes.html)
```sql
SELECT x1.table_in_trouble, pg_relation_size(x1.table_in_trouble)/1024/1024  AS sz_n_MB, x1.seq_scan, x1.idx_scan,
CASE 
WHEN pg_relation_size(x1.table_in_trouble) > 500000000  THEN 'Exceeds 500 megs, too large to count in a view. For a count, count individually'::text
ELSE count(x1.table_in_trouble)::text
END AS tbl_rec_count, 
x1.priority
FROM ( SELECT (schemaname::text || '.'::text) || relname::text AS table_in_trouble, seq_scan, idx_scan,
CASE
WHEN (seq_scan - idx_scan) < 500 THEN 'Minor Problem'::text
WHEN (seq_scan - idx_scan) >= 500 AND (seq_scan - idx_scan) < 2500 THEN 'Major Problem'::text
WHEN (seq_scan - idx_scan) >= 2500 THEN 'Extreme Problem'::text
ELSE NULL::text
END AS priority
FROM  pg_stat_all_tables
WHERE  seq_scan > idx_scan 
AND schemaname != 'pg_catalog'::name 
AND seq_scan > 100) x1 
GROUP BY x1.table_in_trouble, x1.seq_scan, x1.idx_scan,x1.priority 
ORDER BY  x1.seq_scan desc ,  x1.priority DESC limit 100 ;
```


## 对象管理
### 对象查看
```sql
--查看表大小
select relname, pg_size_pretty(pg_relation_size(relid)) from pg_stat_user_tables order by pg_relation_size(relid) desc; 

--查看索引的大小：
SELECT table_name,
   pg_size_pretty(table_size) AS table_size,
   pg_size_pretty(indexes_size) AS indexes_size,
   pg_size_pretty(total_size) AS total_size
FROM (
SELECT
table_name,
pg_table_size(table_name) AS table_size,
pg_indexes_size(table_name) AS indexes_size,
pg_total_relation_size(table_name) AS total_size
FROM (
SELECT ('"' || table_schema || '"."' || table_name || '"') AS table_name
FROM information_schema.tables
) AS all_tables
ORDER BY total_size DESC
) AS pretty_sizes   limit 10;

--查看表与索引的大小SQL 

SELECT schemaname,tablename,
   pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS size_p,
   pg_total_relation_size(schemaname || '.' || tablename) AS siz,
   pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size_p,
   pg_total_relation_size(schemaname || '.' || tablename) - pg_relation_size(schemaname || '.' || tablename) AS index_size,
   (100*(pg_total_relation_size(schemaname || '.' || tablename) - pg_relation_size(schemaname || '.' || tablename)))/CASE 
    WHEN pg_total_relation_size(schemaname || '.' || tablename) = 0 
   THEN 1 ELSE pg_total_relation_size(schemaname || '.' || tablename) END || '%' AS index_pct
FROM pg_tables
ORDER BY siz DESC LIMIT 50;



select pg_relation_size(oid)/1024/1024 size_mb,relname from pg_class 
where relkind in ('i','t') order by pg_relation_size(oid) desc limit 20;

select * from pg_tables where tablename='white_list'; --查表    
select table_catalog,table_name,column_name,ordinal_position,column_default,is_nullable,data_type,character_maxinum_length 
from information_schema.columns where table_name='white_list' order by ordinal_position; --查表字段
--查索引定义 
select b.indexrelid from pg_class a,pg_index b where a.oid=b.indrelid and a.relname='white_list'; 
--查序列 
select * from information_schema.sequences where sequence_name='t_white_id_seq';
select oid,conname,connamespace,contype from pg_constraint where conname like '%white%'; --查约束 

--查function定义 
select oid from pg_proc where procname='zhprs_start'; 
select * from pg_get_functiondef('oid'); 

--同样的，可以通过系统表信息函数，来获取对象的创建语句 
pg_get_viewdef(view_oid) 
pg_get_ruledef(rule_oid) 
pg_get_indexdef(index_oid) 
pg_get_triggerdef(trigger_oid) 
pg_get_constraintdef(constraint_oid) 

--查询库中所有分区表子表个数
SELECT nspname,relname,COUNT(*) AS partition_num  
FROM   pg_class c ,pg_namespace n ,pg_inherits i
WHERE  c.oid = i.inhparent
      AND c.relnamespace = n.oid
      AND c.relhassubclass
      AND c.relkind = 'r'
GROUP BY 1,2 ORDER BY partition_num DESC;
--查询指定分区表信息
SELECT
    nmsp_parent.nspname AS parent_schema ,
    parent.relname AS parent ,
    nmsp_child.nspname AS child ,
    child.relname AS child_schema
FROM
    pg_inherits JOIN pg_class parent
        ON pg_inherits.inhparent = parent.oid JOIN pg_class child
        ON pg_inherits.inhrelid = child.oid JOIN pg_namespace nmsp_parent
        ON nmsp_parent.oid = parent.relnamespace JOIN pg_namespace nmsp_child
        ON nmsp_child.oid = child.relnamespace
WHERE
    parent.relname = 'table_name';

--查看函数调用 次数

select * from pg_stat_user_functions order by calls desc;
```



## 参考
[德哥的PostgreSQL私房菜 - 史上最屌PG资料合集](https://yq.aliyun.com/articles/59251?spm=5176.100239.bloglist.95.5S5P9S)