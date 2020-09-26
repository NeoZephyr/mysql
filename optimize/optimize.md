## 分页
```sql
select * from `order` order by order_no limit 10000, 20;
```

```sql
select * from `order` where id > (
    select id from `order` order by order_no limit 10000, 1
) limit 20;
```

## innodb
1. innodb_buffer_pool_size

设置得越大，InnoDB 表性能就越好。但设置得过大可能会导致系统发生 SWAP 页交换。所以需要在 IBP 大小和其它系统服务所需内存大小之间取得平衡

默认的内存大小是 128M，推荐配置为服务器物理内存的 80%

通过计算 InnoDB 缓冲池的命中率来调整 IBP 大小：
(1 - innodb_buffer_pool_reads / innodb_buffer_pool_read_request) * 100

如果将 IBP 的大小设置为物理内存的 80% 以后，发现命中率还是很低，就应该考虑扩充内存来增加 IBP 的大小

2. innodb_buffer_pool_instances

InnoDB 中的 IBP 缓冲池被划分成了多个实例，对于具有数千兆字节的缓冲池的系统来说，将缓冲池划分为单独的实例可以减少不同线程读取和写入缓存页面时的争用，从而提高系统的并发性。该参数项仅在将 innodb_buffer_pool_size 设置为 1GB 或更大时才会生效

如果 innodb_buffer_pool_size 大小超过 1GB， innodb_buffer_pool_instances 值就默认为 8；否则，默认为 1

建议指定 innodb_buffer_pool_instances 的大小，并保证每个缓冲池实例至少有 1GB 内存。通常，建议 innodb_buffer_pool_instances 的大小不超过 innodb_read_io_threads + innodb_write_io_threads 之和，建议实例和线程数量比例为 1:1

3. innodb_read_io_threads / innodb_write_io_threads

在默认情况下，MySQL 后台线程包括了主线程、IO 线程、锁线程以及监控线程等，其中读写线程属于 IO 线程，主要负责数据库的读取和写入操作，这些线程分别读取和写入 innodb_buffer_pool_instances 创建的各个内存页面

MySQL 支持配置多个读写线程， 即通过 innodb_read_io_threads 和 innodb_write_io_threads 设置读写线程数量。读写线程数量值默认为 4，也就是总共有 8 个线程同时在后台运行

通过以下查询来确定读写比率：
```sql
-- 读
SHOW GLOBAL STATUS LIKE 'Com_select';

-- 写
SHOW GLOBAL STATUS WHERE Variable_name IN ('Com_insert', 'Com_update', 'Com_replace');
```

如果读大于写，考虑将读线程的数量设置得大一些，写线程数量小一些；否则，反之

4. innodb_log_file_size

InnoDB 中执行的每个写入查询都会在日志文件中获得重做条目，以便在发生崩溃时可以恢复更改。当日志文件大小已经超过参数设置的日志文件大小时，InnoDB 会自动切换到另外一个日志文件，由于重做日志是一个循环使用的环，在切换时，就需要将新的日志文件脏页的缓存数据刷新到磁盘中

理论上来说，innodb_log_file_size 设置得越大，缓冲池中需要的检查点刷新活动就越少，从而节省磁盘 I/O。但是，如果日志文件设置得太大，恢复时间就会变长，这样不便于 DBA 管理。在大多数情况下，将日志文件大小设置为 1GB 就足够了

5. innodb_log_buffer_size

InnoDB 的更新操作采用的是 Write Ahead Log 策略，即先写日志，再写入磁盘。当一条记录更新时，InnoDB 会先把记录写入到 redo log buffer 中，并更新内存数据

这个参数决定了 InnoDB 重做日志缓冲池的大小，默认值为 8MB。如果高并发中存在大量的事务，该值设置得太小，就会增加写入磁盘的 I/O 操作。可以通过增大该参数来减少写入磁盘操作，从而提高并发时的事务性能

6. innodb_flush_log_at_trx_commit

这个参数可以控制重做日志从 redo log buffer 刷新到磁盘中的策略

当设置该参数为 0 时，InnoDB 每秒种就会触发一次缓存日志写入到文件中并刷新到磁盘的操作，这有可能在数据库崩溃后，丢失 1s 的数据

当设置该参数为 1 时，则表示每次事务的 redo log 都会直接持久化到磁盘中，这样可以保证 MySQL 异常重启之后数据不会丢失。默认值为 1

当设置该参数为 2 时，每次事务的 redo log 都会直接写入到文件中，再将文件刷新到磁盘

7. max_connections

允许连接到 MySQL 数据库的最大连接数量，默认为 151

8. back_log

TCP 连接请求排队等待栈，并发量比较大的情况下，可以适当调大该参数，增加短时间内处理连接请求量

9. thread_cache_size

MySQL 接收到客户端请求时，需要生成线程用于处理连接。当连接断开时，线程并不会立刻销毁，而是对线程进行缓存，便于下一个连接使用，减少线程的创建和销毁

如果状态值过大，说明 MySQL 一直在创建处理连接的线程，可以适当调大 thread_cache_size
```sql
SHOW GLOBAL STATUS LIKE 'Threads_created';
```

