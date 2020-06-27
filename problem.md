## 大量短连接
当数据库处理得慢的时候，连接数就会暴涨
```sql
show variables like "%max_connections%";
```
面对这种情况，通常处理掉占着连接但不工作的线程
```sql
show processlist;
select * from information_schema.innodb_trx;
```
查找 sleep 状态的线程，且不在事务中
```sql
kill connection_id;
```
从数据库端主动断开连接，客户端并不会马上知道，而是要等到客户端在发起下一个请求的时候，才会收到报错

## 查询慢
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=100000)do
    insert into t values(i,i);
    set i=i+1;
  end while;
end;;
delimiter ;

call idata();
```

1. 等待 MDL 锁
```sql
select * from t where id=1;
```
```sql
-- Waiting for table metadata lock
show processlist;
```
当有一个线程正在表 t 上请求或者持有 MDL 写锁，把 select 语句阻塞了。示例如下：
sessionA
```sql
-- 持有 MDL 写锁
lock table t write;
```
sessionB
```sql
-- 获取 MDL 读锁
select * from t where id = 1;
```

解决办法就是找到加锁的线程，将其 kill 掉。如果设置了参数 `performance_schema` 为 ON
```sql
show variables like "%performance_schema%";
```
可以找到持有 MDL 写锁的线程 id
```sql
select blocking_pid from sys.schema_table_lock_waits;
```

2. 等待 flush

3. 等待行锁
sessionA
```sql
begin;
update t set c = c + 1 where id = 1;
```
sessionB
```sql
select * from t where id = 1 lock in share mode;
```

```sql
select * from sys.innodb_lock_waits where locked_table='`test`.`t`';
```

4. 其它
```sql
-- b 字段有所以，且定义为 varchar(10)
-- 假设有 10 万条数据的 b 值为 1234567890
-- 查询时，会截取前 10 个字节，传给存储引擎做匹配，于是查出了 10 万条数据，然后做了 10 万次回表
-- 每次回表之后查出的整行到 server 层判断
-- 最终返回空值
select * from t where b='1234567890abcd';
```

## 判断数据库状态
```sql
-- innodb 最多允许并发线程数量，超过并发线程数量时，新的请求就会被 blocked 住
-- 默认为 0，表示不限制并发线程数量
-- 建议把 innodb_thread_concurrency 设置为 64 - 128 之间
show variables like "%innodb_thread_concurrency%";
```
并发连接可以通过以下命令查看，一般达到几千问题不大
```sql
show processlist;
```
并发查询通过参数 `innodb_thread_concurrency` 限制，由于并发查询会真正地占有 cpu，所以需要限制。当并发线程数超过 `innodb_thread_concurrency` 之后，系统就会阻塞。需要注意的是，当线程进入锁等待之后，并发线程计数会减一，也就是说等行锁（也包括间隙锁）的线程是不算在并发线程计数中，只有真正在占有 cpu 的线程才会计数

当并发线程超过 `innodb_thread_concurrency` 之后，系统已经阻塞，但使用 `select 1` 检测系统还是正常的

使用以下语句，可以检测出由于并发线程过多导致的数据库不可用的情况
```sql
select * from mysql.health_check; 
```

如果出现更新事务写 `binlog`，而 `binlog` 所在磁盘的空间占用率达到 100%，此时所有的更新语句和事务提交的 commit 语句就都会被堵住。但是，系统这时候还是可以正常读数据的，使用上面的语句检查系统还是正常的

```sql
update mysql.health_check set t_modified=now();
```

主库与备库都应该进行检查
```sql
CREATE TABLE `health_check` (
  `id` int(11) NOT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

/* 检测命令 */
insert into mysql.health_check(id, t_modified) values (@@server_id, now()) on duplicate key update t_modified=now();
```

用于使用 `update` 方法需要资源很少，可能系统 IO 资源已经不足，但是还能执行成功，这样就造成了发现系统不正常太慢，不能及时切换主备

MySQL 5.6 版本以后提供的 performance_schema 库，就在 file_summary_by_event_name 表里统计了每次 IO 请求的时间
```sql
select * from performance_schema.file_summary_by_event_name;
```

如果打开所有的 performance_schema 项，比较影响性能，可以通过下面的方法打开或者关闭某个具体项的统计
```sql
update performance_schema.setup_instruments set ENABLED='YES', Timed='YES' where name like '%wait/io/file/innodb/innodb_log_file%';
```

设定阈值，单次 IO 请求时间超过 200 毫秒属于异常
```sql
select event_name,MAX_TIMER_WAIT FROM performance_schema.file_summary_by_event_name where event_name in ('wait/io/file/innodb/innodb_log_file','wait/io/file/sql/binlog') and MAX_TIMER_WAIT>200*1000000000;
```
发现异常后，取到你需要的信息，然后把之前的统计信息清空
```sql
truncate table performance_schema.file_summary_by_event_name;
```

## 数据恢复
### 误删行
用 Flashback 工具，修改 `binlog` 的内容，拿回原库重放。使用这个方案的前提是，需要确保 `binlog_format=row` 和 `binlog_row_image=FULL`

### 误删库 / 表
假如有人中午 12 点误删了一个库，恢复数据的流程如下：
1. 取最近一次全量备份，假设这个库是一天一备，上次备份是当天 0 点
2. 用备份恢复出一个临时库
3. 从日志备份里面，取出凌晨 0 点之后的日志
4. 把这些日志，除了误删除数据的语句外，全部应用到临时库

为了加速数据恢复，如果这个临时库上有多个数据库，可以在使用 `mysqlbinlog` 命令时，加上 `–database` 参数，用来指定误删表所在的库

在应用日志的时候，需要跳过 12 点误操作的那个语句的 `binlog`。如果原实例没有使用 GTID 模式，只能在应用到包含 12 点的 `binlog` 文件的时候，先用 `–stop-position` 参数执行到误操作之前的日志，然后再用 `–start-position` 从误操作之后的日志继续执行；如果实例使用了 GTID 模式，且误操作命令的 GTID 是 gtid1，那么只需要执行 `set gtid_next=gtid1;begin;commit;`，把这个 GTID 加到临时实例的 GTID 集合，之后按顺序执行 `binlog` 的时候，就会自动跳过误操作的语句

还可以搭建延迟复制备库，通过 `CHANGE MASTER TO MASTER_DELAY = N` 命令，指定这个备库持续保持跟主库有 N 秒的延迟。比如将 N 设置为 3600，表示如果主库上有数据被误删了，并且在 1 小时内发现了这个误操作命令，这个命令就还没有在这个延迟复制的备库执行。这时候到这个备库上执行 `stop slave`，再通过之前介绍的方法，跳过误操作命令，就可以恢复出需要的数据，缩短了恢复需要的时间

### 预防误删库 / 表
1. 权限隔离
2. 在删除数据表之前，必须先对表做改名操作（加 `_to_be_deleted` 后缀，且只允许删除固定后缀的表）。观察一段时间，确保对业务无影响以后再删除这张表

## kill 语句
`kill + query + thread_id` 表示终止这个线程中正在执行的语句
`kill + connection + thread_id` 表示断开这个线程的连接

当用户执行 `kill query thread_id_B` 时，mysql 处理 kill 命令：
1. 把 session B 的运行状态改成 THD::KILL_QUERY
2. 给 session B 的执行线程发一个信号

kill 可能会无效，有以下原因：
1. 线程没有执行到判断线程状态的逻辑，或者由于 IO 压力比较大，导致不能及时判断线程的状态
2. 终止逻辑耗时较长，常见的情况是：超大事务执行期间被 kill；大查询回滚；DDL 命令执行到最后阶段被 kill

## 全表扫描
```sql
mysql -h$host -P$port -u$user -p$pwd -e "select * from db1.t" > $target_file
```
服务端并不需要保存一个完整的结果集，取数据和发数据的流程是这样的：
1. 获取一行，写到 `net_buffer` 中。这块内存的大小是由参数 `net_buffer_length` 定义的，默认是 16k
2. 重复获取行，直到 `net_buffer` 写满，调用网络接口发出去
3. 如果发送成功，就清空 `net_buffer`，然后继续取下一行，并写入 `net_buffer`
4. 如果发送函数返回 EAGAIN 或 WSAEWOULDBLOCK，就表示本地网络栈（socket send buffer）写满，进入等待。直到网络栈重新可写，再继续发送

可以看出，mysql 是边读边发的。如果客户端接收得慢，会导致 MySQL 服务端由于结果发不出去，这个事务的执行时间变长。如果通过 `show processlist` 查看执行状态处于 "Sending to client" 的状态，就表示服务器端的网络栈写满了。这时候，就需要优化查询结果，并评估这么多的返回结果是否合理

InnoDB Buffer Pool 的大小是由参数 `innodb_buffer_pool_size` 确定的，一般建议设置成可用物理内存的 60%~80%。InnoDB 内存管理用的是最近最少使用 (Least Recently Used, LRU) 算法，这个算法的核心就是淘汰最久未使用的数据。在 InnoDB 实现上，按照 5:3 的比例把整个 LRU 链表分成了 young 区域和 old 区域。这样在全表扫描时，需要新插入的数据页，都被放到 old 区域。当扫描完这个数据页的数据之后，再不会用到，于是始终没有机会移到链表头部（也就是 young 区域），很快就会被淘汰出去。这样，保证了 Buffer Pool 响应正常业务的查询命中率

