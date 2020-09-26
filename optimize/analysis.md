## 慢查询定位
开启慢查询
```sql
show variables like '%slow_query_log';
```
```sql
set global slow_query_log='ON';
```

设置慢查询的时间阈值
```sql
show variables like '%long_query_time%';
```
```sql
set global long_query_time = 3;
```

使用 mysqldumpslow 工具统计慢查询日志，命令的具体参数如下
1. -s：采用 order 排序的方式，排序方式可以有以下几种。分别是 c（访问次数）、t（查询时间）、l（锁定时间）、r（返回记录）、ac（平均查询次数）、al（平均锁定时间）、ar（平均返回记录数）和 at（平均查询时间）。其中 at 为默认排序方式
2. -t：返回前 N 条数据
3. -g：正则表达式


## 查看执行计划
SQL 执行的顺序是根据 id 从大到小执行的，当 id 相同时，从上到下执行

数据表的访问类型所对应的 type 列可能有以下几种情况：
1. all，全数据表扫描
2. index，全索引表扫描
3. range，对索引列进行范围查询
4. index_merge，合并索引，使用多个单列索引搜索
5. ref，根据索引查找一个或多个值
6. eq_ref，搜索时使用 primary key 或 unique 类型，常用于多表联查
7. const, 常量，表最多有一个匹配行，因为只有一行，在这行的列值可被优化器认为是常数
8. system，系统，表只有一行

all 是最坏的情况，因为采用了全表扫描的方式。index 和 all 差不多，只不过 index 对索引表进行全扫描。如果在 Extral 列中看到 Using index，说明采用了索引覆盖，也就是索引可以覆盖所需的 SELECT 字段，就不需要进行回表，减少了数据查找的开销
```sql
# 联合索引 composite_index (user_id, comment_text)
EXPLAIN SELECT comment_id, comment_text, user_id FROM product_comment 
```

range 表示采用了索引范围扫描，从这一级别开始，索引的作用会越来越明显，因此尽量让 SQL 查询可以使用到 range 这一级别及以上的 type 访问方式

index_merge 说明查询同时使用了两个或以上的索引，最后取了交集或者并集
```sql
# comment_id 为主键，user_id 是普通索引
EXPLAIN SELECT comment_id, product_id, comment_text, user_id FROM product_comment
WHERE comment_id = 500000 OR user_id = 500000
```

ref 类型表示采用了非唯一索引，或者是唯一索引的非唯一性前缀
```sql
# user_id 为普通索引
EXPLAIN SELECT comment_id, comment_text, user_id FROM product_comment WHERE user_id = 500000;
```

eq_ref 类型是使用主键或唯一索引时产生的访问方式，通常使用在多表联查中
```sql
EXPLAIN SELECT * FROM product_comment JOIN user WHERE product_comment.user_id = user.user_id;
```

const 类型表示使用了主键或者唯一索引（所有的部分）与常量值进行比较
```sql
EXPLAIN SELECT comment_id, comment_text, user_id FROM product_comment WHERE comment_id = 500000;
```

system 类型一般用于 MyISAM 或 Memory 表，属于 const 类型的特例，当表只有一行时连接类型为 system


## 查看执行成本
默认情况下，profiling 是关闭的，可以在会话级别开启这个功能
```sql
show variables like 'profiling';
```
```sql
set profiling = 'ON';
```

查看当前会话的 profiles
```sql
show profiles;
```

查看上一个查询的开销
```sql
show profile;
```

查看指定的 Query ID 的开销
```sql
show profile for query 2;
```
查看不同部分的开销
```sql
show profile cpu, block io for query 2;
```

SHOW PROFILE 命令将被弃用，我们可以从 information_schema 中的 profiling 数据表进行查看

