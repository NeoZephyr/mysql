## 执行计划
### id
每个执行计划都有一个 id，如果是一个联合查询，这里还将有多个 id

### select_type
表示 SELECT 查询类型，常见的有
SIMPLE：普通查询，即没有联合查询、子查询
PRIMARY：主查询
UNION：UNION 中后面的查询
SUBQUERY：子查询

### table
当前执行计划查询的表，如果有别名，则显示别名信息

### partitions
访问的分区表信息

### type
表示从表中查询到行所执行的方式，查询方式是 SQL 优化中一个很重要的指标，结果值从好到差依次是：
system > const > eq_ref > ref > range > index > ALL

system/const：表中只有一行数据匹配，此时根据索引查询一次就能找到对应的数据
```sql
select * from order where id = 10;
```

eq_ref：使用唯一索引扫描，常见于多表连接中使用主键和唯一索引作为关联条件
```sql
select * from order, order_detail where order.id = order_detail.order_id;
```

ref：非唯一索引扫描，还可见于唯一索引最左原则匹配扫描
```sql
select * from order where order_no = "xxx";
```

range：索引范围扫描，比如，<，>，between 等操作
```sql
select * from order where id > 10;
```

index：索引全表扫描，遍历整个索引树
```sql
select id from order;
```

ALL：表示全表扫描，需要遍历全表来找到对应的行
```sql
select * from order where pay_money = 0;
```

### possible_keys
可能使用到的索引

### key
实际使用到的索引

### key_len
当前使用的索引的长度

### ref
关联 id 等信息

### rows
查找到记录所扫描的行数

### filtered
查找到所需记录占总扫描记录数的比例

### Extra
额外的信息


## Profile
查询是否支持
```sql
select @@have_profiling
```

```sql
show profiles;
```


## 慢查询
```sql
Show variables like 'slow_query%';
Show variables like 'long_query_time';
```

```sql
-- 开启慢 SQL 日志
set global slow_query_log='ON';

-- 记录日志地址
set global slow_query_log_file='/var/lib/mysql/test-slow.log';

-- 最大执行时间
set global long_query_time=1;
```
