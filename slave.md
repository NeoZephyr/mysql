## 主从同步作用
1. 提高数据库的吞吐量
2. 读写分离，提高数据库并发处理能力
3. 数据备份
4. 具有高可用性，当主库出现故障或宕机的情况下，可以切换到从库上，保证服务的正常运行


## 主从同步原理
主从复制过程中，基于 3 个线程来操作，一个主库线程，两个从库线程

二进制日志转储线程（Binlog dump thread）是一个主库线程。当从库线程连接的时候，主库可以将二进制日志发送给从库，当主库读取事件的时候，会在 Binlog 上加锁，读取完成之后，再将锁释放掉

从库 I/O 线程会连接到主库，向主库发送请求更新 Binlog。这时从库的 I/O 线程就可以读取到主库的二进制日志转储线程发送的 Binlog 更新部分，并且拷贝到本地形成中继日志（Relay log）

从库 SQL 线程会读取从库中的中继日志，并且执行日志中的事件，从而将从库中的数据与主库保持同步


## 数据一致性
### 异步复制
客户端提交 COMMIT 之后不需要等从库返回任何结果，而是直接将结果返回给客户端，这样做不会影响主库写的效率，但可能会存在主库宕机，而 Binlog 还没有同步到从库的情况。这时从从库中选择一个作为新主，那么新主则可能缺少原来主服务器中已提交的事务

### 半同步复制
MySQL5.5 版本之后开始支持半同步复制的方式。客户端提交 COMMIT 之后不直接将结果返回给客户端，而是等待至少有一个从库接收到了 Binlog，并且写入到中继日志中，再返回给客户端。这样提高了数据的一致性，但降低了主库写的效率。在 MySQL5.7 版本中还增加了一个 rpl_semi_sync_master_wait_for_slave_count 参数，可以对应答的从库数量进行设置，默认为 1，也就是说只要有 1 个从库进行了响应，就可以返回给客户端

### 组复制
组复制技术，简称 MGR（MySQL Group Replication）。是 MySQL 在 5.7.17 版本中推出的一种新的数据复制技术，这种复制技术是基于 Paxos 协议的状态机复制。首先将多个节点共同组成一个复制组，在执行读写事务的时候，需要通过一致性协议层的同意。而只读事务则不需要经过组内同意，直接 COMMIT 即可。在一个复制组内有多个节点组成，它们各自维护了自己的数据副本，并且在一致性协议层实现了原子消息和全局有序消息，从而保证组内数据的一致性


## 日志同步流程
1. 在备库上通过 `change master` 命令，设置主库的 IP、端口、用户名、密码，以及 `binlog` 文件名和日志偏移量
2. 在备库上面执行 `start slave` 命令，这时候备库会启动两个线程，即 io_thread 与 sql_thread，其中 io_thread 负责与主库建立连接
3. 主库校验完用户名、密码后，开始按照备库传过来的位置，从本地读取 `binlog` 发给备库
4. 备库拿到 `binlog` 后，写到本地文件，称为中转日志 `relay log`
5. sql_thread 读取中转日志，解析出日志里的命令执行

## `binlog` 格式
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `t_modified`(`t_modified`)
) ENGINE=InnoDB;

insert into t values(1,1,'2018-11-13');
insert into t values(2,2,'2018-11-12');
insert into t values(3,3,'2018-11-11');
insert into t values(4,4,'2018-11-10');
insert into t values(5,5,'2018-11-09');
```
执行以下语句
```sql
delete from t where a>=4 and t_modified<='2018-11-10' limit 1;
```

当 binlog_format=statement 时，`binlog` 里面记录的就是 SQL 语句的原文
```sql
show binlog events in 'master.000001';
```
BEGIN 跟 COMMIT 对应，表示中间是一个事务，在 BEGIN 与 COMMIT 中间就是真实的执行语句

当 binlog_format=row 时，与 statement 格式一样，BEGIN 跟 COMMIT 对应，表示中间是一个事务。但是事务中间没有 SQL 语句原文，替换成了两个 event: Table_map 和 Delete_rows

借助 mysqlbinlog 工具，解析和查看 `binlog` 中的内容
```sql
mysqlbinlog  -vv data/master.000001 --start-position=8900;
```
可以得到以下信息：
server id 1，表示这个事务是在 server_id = 1 这个库上面执行的
由于 -vv 参数是为了把内容都解析出来，可以看到各个字段的值，例如 @1, @2, @3 等
Xid 用于表示事务被正确地提交了

当 binlog_format 使用 row 格式的时候，`binlog` 里面记录了真实删除行的主键 id，这样 `binlog` 传到备库去的时候，就肯定会删除 id=4 的行，不会有主备删除不同行的问题

因为有些 statement 格式的 `binlog` 可能会导致主备不一致，所以要使用 row 格式。但是 row 格式很占空间，例如 delete 掉 10 万行数据，在使用 row 格式的情况下，就需要把这 10 万条记录都写到 `binlog` 中。于是就有了 mixed 格式的 `binlog`，如果执行的 sql 语句会引起主备不一致，就使用 row 格式，否则就使用 statement 格式

将 master.000001 文件里面从第 2738 字节到第 2973 字节中间这段内容解析出来，放到 MySQL 执行
```sql
mysqlbinlog master.000001  --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -uroot -proot;
```

## 循环复制
如果主备使用的是双 M 结构，业务逻辑在节点 A 上面更新了一条语句，然后将生产的 `binlog` 发给节点 B，节点 B 执行完这条更新语句之后也会生产 `binlog`，这样就会有循环复制问题
```sql
-- 建议设置为 on，备库执行 relay log 后生成 binlog
show variables like "%log_slave_updates%";
```
可以通过设置 server id 来解决两个节点间的循环复制问题：
1. 规定两个库的 server id 必须不同，如果相同，则不能设定为主备关系
2. 备库接到 binlog 并在重放过程中，生成与原 `binlog` 的 `server id` 相同的新的 `binlog`
3. 每个库在收到从自己的主库发过来的日志后，先判断 server id 是否与自己的相同，如果相同则表示是自己生成的，就直接丢弃这个日志

## 主备延迟
主备同步的时间点如下：
1. 主库执行事务，写入 `binlog`，这个时刻记为 T1
2. 备库接收完这个 `binlog`，时刻记为 T2
3. 备库执行完这个事务，时刻记为 T3

同一个事务，主备延迟时间大约就是 T3 - T1，即主库与备库事务完成的时间之间的差值。可以在备库上执行以下命令查看延迟时间：
```sql
-- seconds_behind_master
show slave status;
```

通常来说，网络正常的情况下，T2 - T1 的值非常小，也就是说日志从主库传给备库所需的时间是很短的。那么，主备延迟的主要来源是备库接收完 `binlog` 和执行完这个事务之间的时间差。因此，主备延迟最直接的表现是，备库消费 `relay log` 比主库生产 `binlog` 的速度要慢

主备延迟一般有以下原因：
1. 备库所在机器的性能要比主库所在的机器性能差
主库与备库选择相同规格的机器，并且做对称部署

2. 备库压力大
备库上进行大量分析查询。我们可以选择一主多从，多个从库分担查询压力；然后通过 binlog 输出到外部系统，例如 hadoop 系统，让外部系统提供统计查询能力

3. 大事务
主库上必须等事务执行完成才会写入 `binlog`，再传给备库。比如一次性 delete 删除太多数据；大表 DDL

## 主备切换
### 可靠性优先策略
1. 判断备库的 `seconds_behind_master` 是否小于某个值，如果小于则继续下一步，否则持续重试
2. 将主库改为只读状态，即设置 `readonly` 为 true
3. 判断备库的 `seconds_behind_master`，直到变为 0 为止
4. 将备库改为可读写状态，即设置 `readonly` 为 false
5. 业务请求设置到备库

### 可用性优先策略
将可靠性优先策略中的步骤 4、5 调整到最开始执行，也就是说不等主备数据同步，直接把连接切到备库。这个切换流程，保证了系统几乎没有不可用时间，但是可能会出现数据不一致的情况。使用 row 格式的 `binlog` 时，数据不一致的问题更容易被发现。

## GTID
GTID 的全称是 Global Transaction Identifier，即全局事务 ID，是一个事务在提交的时候生成的，是这个事务的唯一标识。由两个部分组成：
```
GTID=source_id:transaction_id
```
source_id 是一个实例第一次启动时自动生成全局唯一的值
transaction_id 初始值是 1，每次提交事务的时候分配并加 1

如果需要启动 GTID 需要设置以下参数：
```
gtid_mode=on
enforce_gtid_consistency=on
```

在 GTID 模式下，每个事务都会跟一个 GTID 对应。GTID 有两种生成方式，取决于 `session` 变量 `gtid_next` 的值
1. 如果 gtid_next=automatic，记录 `binlog` 的时候，先记录一行 `SET @@SESSION.GTID_NEXT='source_id:transaction_id'`，然后将这个 GTID 加入到本实例的 GTID 集合
2. 如果通过 `set gtid_next='current_gtid'` 指定了 GTID 的值，此时，如果 `current_gtid` 已经存在于实例的 GTID 集合中，则对应事务会被系统忽略；如果 `current_gtid` 没有存在于实例的 GTID 集合中，就将这个 `current_gtid` 分配给接下来要执行的事务，不需要系统为这个事务生成新的 GTID

可以看出，每个实例都维护了一个 GTID 集合，用来对应这个实例执行过的所有事务

基于 GTID 切换主库
```sql
CHANGE MASTER TO 
MASTER_HOST=$host_name 
MASTER_PORT=$port 
MASTER_USER=$user_name 
MASTER_PASSWORD=$password 
master_auto_position=1 
```
`master_auto_position=1` 就表示这个主备关系使用的是 GTID 协议
假设将数据库 B 的主库设置为数据库 A，从库获取 `binlog` 的逻辑是这样的：
1. 基于主备协议建立连接
2. 数据库 B 将自己的 GTID 集合 set_b 发送给数据库 A
3. 数据库 A 计算自己的 GTID 集合 set_a 与 set_b 的差集，即所有存在于 set_a，但是不存在于 set_b 的 GTID 集合
4. 判断数据库 A 本地是否包含这个差集需要的所有 `binlog` 事务
5. 如果不包含，表示数据库 A 已经把数据库 B 需要的 `binlog` 给删掉了，直接返回错误；如果全部包含，从自己的 `binlog` 文件里面，找出第一个不在 set_b 集合的事务发送给数据库 B
6. 之后就从这个事务开始，往后读文件，按顺序取 `binlog` 发送给数据库 B

## 并行复制
客户端写入主库的并发度要远高于备库的 sql_thread 执行 `relay log`  的并发度，如果备库更新数据只是使用单线程，那么在主库并发高、TPS 高时就会出现严重的主备延迟问题，所以需要多线程并行复制。sql_thread 不再直接更新数据，只负责读取中转日志和分发事务给 worker 线程，worker 线程进行真正的更新日志操作，worker 线程个数由参数 `slave_parallel_workers` 决定，通常这个值设置为 8~16 之间最好（32 核物理机的情况）
```sql
show variables like "%slave_parallel_workers%";
```

sql_thread 在进行中转日志的分发时，需要遵循以下原则：
1. 更新同一行的两个事务，必须被分发到同一个 worker 中
2. 同一个事务不能被拆开，必须放到同一个 worker 中

mysql5.7 提供了并行复制功能，并由参数 `slave_parallel_type` 控制并行复制策略：
```sql
-- 配置为 DATABASE，表示使用按库并行策略
-- 配置为 LOGICAL_CLOCK
show variables like "%slave_parallel_type%";
```

## 过期读
在一主多从的结构中，可能会有通过从库读到系统的一个过期状态。解决这个问题，有以下方案：

### 强制走主库
将查询请求进行分类

1. 对于必须要拿到最新结果的请求，强制将其发到主库上
2. 对于可以读到旧数据的请求，才将其发到从库上

### Sleep
主库更新后，读从库之前先 sleep 一下，执行一条 `select sleep(1)` 命令

### 判断主备无延迟
每次从库执行查询请求前，先判断 `seconds_behind_master` 是否已经等于 0。如果还不等于 0 ，就等到这个参数变为 0 再执行查询请求

采用对比位点判断主备无延迟。执行 `show slave status`，Master_Log_File 和 Read_Master_Log_Pos，表示的是读到的主库的最新位点；Relay_Master_Log_File 和 Exec_Master_Log_Pos，表示的是备库执行的最新位点。如果这两组值相同，则表示从库接收到的日志已经同步完成

还可以对比 GTID 集合确保主备无延迟。执行 `show slave status`，Auto_Position=1 ，表示这对主备关系使用了 GTID 协议；Retrieved_Gtid_Set，是备库收到的所有日志的 GTID 集合；Executed_Gtid_Set，是备库所有已经执行完成的 GTID 集合。如果这两个集合相同，也表示备库接收到的日志都已经同步完成

后面两中判断主备无延迟的的方法，都要比判断 seconds_behind_master 是否为 0 更加准确

### semi-sync
通过对比位点和 GTID 集合判断主备是否延迟，判断的是备库收到的日志都执行完成了。如果出现客户端收到提交确认，而备库还没有收到日志，按照之前的逻辑，判断从库已经没有同步延迟，但实际上还是有延迟的

要解决这个问题，引入半同步复制，也就是 semi-sync replication。semi-sync 做了如下设计：
1. 事务提交的时候，主库把 `binlog` 发给从库
2. 从库收到 `binlog` 以后，发回给主库一个 ack，表示收到
3. 主库收到这个 ack 以后，才能给客户端返回“事务完成”的确认

这样，所有给客户端发送过确认的事务，都确保了备库已经收到了这个日志

### 等主库位点
使用 semi-sync 仍然存在以下两个问题：
1. 一主多从的时候，在某些从库执行查询请求会存在过期读的现象
2. 在业务高峰期，持续延迟的情况下，可能出现过度等待的问题

```sql
select master_pos_wait(file, pos[, timeout]);
```
在从库上面执行以上命令，参数 file 和 pos 分别代表主库上面的文件名和位置；timeout 可以选择，设置为 1 则表示函数最多等到 1 秒。该函数有以下可能的返回值：
1. 返回正整数，表示从命令开始执行，到应用完 file 和 pos 表示的 `binlog` 位置，执行的事务数量
2. 返回 null，表示执行期间，备库同步线程发生异常
3. 返回 -1，表示等待超时
4. 返回 0，刚开始执行的时候，就发现已经执行过这个位置了

### GTID 方案
```sql
select wait_for_executed_gtid_set(gtid_set, 1);
```
这个函数有以下返回值：
1. 返回 0，等待，直到这个库执行的事务中包含传入的 gtid_set
2. 返回 1，等待超时

此时，等 GTID 的执行流程如下：
1. trx1 事务更新完成后，从返回包直接获取这个事务的 GTID，记为 gtid1
2. 选定一个从库执行查询语句
3. 在从库上执行 `select wait_for_execute_gtid_set(gtid1, 1)`
4. 如果返回值是 0，则在这个从库执行查询语句
5. 否则，到主库执行查询语句

要在返回包直接获取事务的 GTID，还需要 将参数 `session_track_gtids` 设置为 OWN_GTID，然后通过 API 接口 `mysql_session_track_get_first` 从返回包解析出 GTID 的值即可




