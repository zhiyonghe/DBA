# postgres SQL 优化
## 查找慢SQL
使用pg_stats_statements查找    

开启auto_explain    
使用auto_explain  mode,开启以下选项    
log_nested_statements   
log_min_duration
## Index Tuning
pg_stats... 相关视图    

``` sql
SELECT
  relname,
  seq_scan - idx_scan AS too_much_seq,
  CASE
    WHEN
      seq_scan - coalesce(idx_scan, 0) > 0
    THEN
      'Missing Index?'
    ELSE
      'OK'
  END,
  pg_relation_size(relname::regclass) AS rel_size, seq_scan, idx_scan
FROM
  pg_stat_all_tables
WHERE
  schemaname = 'public'
  AND pg_relation_size(relname::regclass) > 80000
ORDER BY
  too_much_seq DESC;
```
### 没有使用索引
```sql
SELECT
  indexrelid::regclass as index,
  relid::regclass as table,
  'DROP INDEX ' || indexrelid::regclass || ';' as drop_statement
FROM
  pg_stat_user_indexes
  JOIN
    pg_index USING (indexrelid)
WHERE
  idx_scan = 0
  AND indisunique is false;
```
### 统计信息
hash join 需要有足够的内存    
Sequential Scans 适用于数据量少的表（开发 ） 