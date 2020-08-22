--查询索引详细信息
SELECT
  t.tablename,
  indexname,
  c.reltuples AS num_rows,
  pg_size_pretty(pg_relation_size(quote_ident(t.tablename)::text)) AS table_size,
  pg_size_pretty(pg_relation_size(quote_ident(indexrelname)::text)) AS index_size,
  CASE WHEN indisunique THEN 'Y'
    ELSE 'N'
  END AS UNIQUE,
  idx_scan AS number_of_scans,
  idx_tup_read AS tuples_read,
  idx_tup_fetch AS tuples_fetched
FROM pg_tables t
  LEFT OUTER JOIN pg_class c ON t.tablename=c.relname
  LEFT OUTER JOIN
    ( SELECT c.relname AS ctablename, ipg.relname AS indexname, x.indnatts AS number_of_columns, idx_scan, idx_tup_read, idx_tup_fetch, indexrelname, indisunique FROM pg_index x
      JOIN pg_class c ON c.oid = x.indrelid
      JOIN pg_class ipg ON ipg.oid = x.indexrelid
      JOIN pg_stat_all_indexes psai ON x.indexrelid = psai.indexrelid )
    AS foo
  ON t.tablename = foo.ctablename
WHERE t.schemaname='public'
ORDER BY 1,2;


--查询整库索引大小并排序
SELECT c.relname,c2.relname, c2.relpages*8/1024 as size_MB, indexdef||';' as index_def
FROM pg_class c, pg_class c2, pg_index i,pg_indexes iv 
WHERE  c.oid = i.indrelid AND c2.oid = i.indexrelid and c2.relname=iv.indexname ORDER BY c2.relpages*8 desc;

--查看索引是valid 还是ready
说明：
The indisvalid and indisready are only meaningful for indexes created using concurrently ---while they are being created or if creation fails. 
 Once they are successfully created those columns have no meaning any more .
 indisvalid indicates whether the index will be used when querying. indisready indicates whether it gets updated on table modifications. 
 You can set them explicitely if you have the correct access rights

SELECT
    trel.relname AS table_name,
    irel.relname AS index_name,
    string_agg(a.attname, ', ' ORDER BY c.ordinality) AS columns
FROM pg_index AS i
         JOIN pg_class AS trel ON trel.oid = i.indrelid
         JOIN pg_class AS irel ON irel.oid = i.indexrelid
         JOIN pg_attribute AS a ON trel.oid = a.attrelid
         JOIN LATERAL unnest(i.indkey)
    WITH ORDINALITY AS c(colnum, ordinality)
              ON a.attnum = c.colnum
WHERE i.indisvalid -- WHERE not i.indisvalid
GROUP BY i, trel.relname, irel.relname;


--生成删除主键sql
select 'alter table '||t.tablename||' drop CONSTRAINT '||i.indexname||';' from  pg_indexes i ,pg_tables t 
where i.schemaname=t.schemaname and i.tablename=t.tablename  and i.indexname like '%pk%';

--生成索引删除语句
select 'drop index  ' ||i.indexname||';'  from pg_indexes i ,pg_tables t where i.schemaname=t.schemaname and i.tablename=t.tablename;

--生成索引创建语句
select  indexdef||';' from pg_indexes i ,pg_tables t where i.schemaname=t.schemaname and i.tablename=t.tablename;

--生成添加主键语句
select 'alter table ' ||t.tablename||'   add primary key using index '||i.indexname||';' from  pg_indexes i ,pg_tables t 
where i.schemaname=t.schemaname and i.tablename=t.tablename  and i.indexname like '%pk%';


--查询gin索引
 select * from pg_indexes where indexdef like '%gin%';

--生成删除外键
SELECT 'alter table '|| r.conrelid::regclass ||' drop constraint ' ||conname ||';'
FROM pg_catalog.pg_constraint r
WHERE r.contype = 'f' ORDER BY 1;


--生成创建外键的脚本
SELECT 'alter table ' || r.conrelid::regclass  || ' add ' ||pg_catalog.pg_get_constraintdef(r.oid, true) ||';'
FROM pg_catalog.pg_constraint r
WHERE r.contype = 'f' ORDER BY 1;


--生产失效trigger，生效trigger 脚本
select 'alter table ' ||t3.nspname||'.'||t2.relname || ' DISABLE TRIGGER ' || tgname ||';' from pg_trigger t1,pg_class t2,pg_namespace t3 where t1.tgrelid=t2.oid and t2.relnamespace=t3.oid ;
	
select 'alter table ' ||t3.nspname||'.'||t2.relname || ' ENABLE TRIGGER ' || tgname ||';' from pg_trigger t1,pg_class t2,pg_namespace t3 where t1.tgrelid=t2.oid and t2.relnamespace=t3.oid ;
	
--查询trigger状态
select t2.relname,tgname,tgenabled from pg_trigger t1,pg_class t2 where t1.tgrelid=t2.oid ;