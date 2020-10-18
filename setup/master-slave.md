## 启用慢查询日志
```sql
set global slow_query_log_file=slow_log.log;
set global log_queries_not_using_indexes=on;
set global long_query_time=0.001;
set global slow_query_log=on;
```

## 主数据库配置
```sh
log_bin = /data/mysql/sql_log/mysql-bin
server_id = 100
```

## 从数据库配置
```sh
log_bin = /data/mysql/sql_log/mysql-bin
server_id = 101
relay_log = /data/mysql/sql_log/relay-bin
read_only = on
```

## 初始化 slave 数据
1. `--single-transaction` 通过事务保证一致性，保证所有数据一致
2. `-l, --lock-tables` 备份过程中依次锁定每个数据库下的所有表，保证某一数据库下的数据一致，与 `--single-transaction` 互斥
3. `-x, --lock-all-tables` 备份过程中对所有数据库下的表进行加锁
```sql
mysqldump --master-data=2 -uroot -p -A --single-transaction -R --triggers > all.sql
mysql -uroot -p < all.sql
```

## 启动基于日志点的复制链路
```sql
create user 'repl'@'localhost' identified by 'password';
grant replication slave on *.* to 'repl'@'localhost' identified by '123456';
```
```sql
change master to
  master_host = 'master_host_ip',
  master_user = 'repl',
  master_password = 'password',
  master_log_file = 'mysql_log_file_name',
  master_log_pos = '';

start slave;
show slave status;
```