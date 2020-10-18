## 写入逻辑
事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 binlog 文件中。一个事务的 binlog 是不能被拆开的，不论这个事务多大，都要确保一次性写入。每个线程都拥有自己的 binlog cache，大小由参数 binlog_cache_size 控制

```sql
show variables like "%binlog_cache_size%";
```

事务提交的时候，执行器把 binlog cache 里的完整事务写入到 binlog 中，并清空 binlog cache。写入 binlog 持久化由参数 sync_binlog 控制：
```sql
-- 0，表示每次提交事务都把日志写入到 page cache，并且持久化到磁盘
-- 1，表示每次提交事务都持久化到磁盘
-- n，表示每次提交事务都把日志写入到 page cache，但积累到 n 个事务后才持久化到磁盘
show variables like "sync_binlog";
```
建议设置为 1，保证 MySQL 异常重启之后 binlog 不丢失

binlog 的默认是保持时间由参数 expire_logs_days 配置
