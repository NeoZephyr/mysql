## ACID
1. Atomicity：原子性
2. Consistency：一致性
3. Isolation：隔离性
4. Durability：持久性

## 隔离级别
### READ UNCOMMITED
1. 普通 `select` 使用快照读，不加锁。该隔离级别下，没有视图概念，读取数据时直接返回记录上的最新值
```sql
select * from user where id = 1;
```

### READ COMMITED
1. 普通 `select` 使用快照读，不加锁。该隔离级别下，快照读总是能读取到已提交事务写入的数据，即事务在每次读操作时，都会建立 Read View
```
A1: start transaction;
B1: start transaction;
A2: select * from user;
B2: insert into user values (4, 'jack');
A3: select * from user;
B3: commit;
A4: select * from user;
```
```
A2 读到的结果集是 {1, 2, 3}
A3 读到的结果集也是 {1, 2, 3}，因为 事务 B 还没有提交
A4 读到的结果集是 {1, 2, 3, 4}，因为事务 B 已经提交
```

2. 加锁 `select` 以及 `update`、`delete`，除了在外键约束检查以及重复键检查时会封锁区间，其他时刻都只使用记录锁
```sql
select * from user where id = 1 for update;
select * from user where id = 1 in share mode;
```

### REPEATABLE READ
1. 普通 `select` 使用快照读，不加锁。该隔离级别下，首次读记录的时间为 T，此后的无法读取到 T 时间之后事务提交写入的数据，即事务在首次读操作时建立 Read View
```
A1: start transaction;
B1: start transaction;
A2: select * from user;
B2: insert into user values (4, 'jack');
A3: select * from user;
B3: commit;
A4: select * from user;
```
```
A2 读到的结果集肯定是 {1, 2, 3}，这是事务 A 的首次读取，假设为时间 T
A3 读到的结果集也是 {1, 2, 3}，因为 事务 B 还没有提交
A4 读到的结果集还是 {1, 2, 3}，因为事务 B 是在时间 T 之后提交的，A4 得读到和 A2 一样的数据
```

2. 加锁的 `select` 以及 `update`、`delete`
```sql
# 在唯一索引上使用唯一查询条件，使用记录锁，不会封锁记录之间的间隔
select * from user where id = 1 for update;
select * from user where id = 1 in share mode;

update user set name = "pain" where id = 1;
```
```sql
# 其他的查询条件和索引条件，会封锁被扫描的索引范围，并使用间隙锁与临键锁，避免索引范围区间插入记录
select * from user where id < 5 for update;
```

>需要说明的是，如果更新的是聚集索引记录，则对应的普通索引记录也会被隐式加锁。这是由 InnoDB 索引的实现机制决定的：普通索引存储 PK 的值，检索普通索引需要扫描聚集索引

3. <b>mysql 默认隔离级别</b>

### SERIALIZABLE
1. 普通 `select` 转换为 `select ... in share mode`，与 `update`、`delete` 操作互斥
```sql
select * from user where id = 1;
```

#### case1
事务 A 先执行，且处于未提交状态；此时由于事务 A 在 `id=1` 记录上面加了行所，因此事务 B 会阻塞
```sql
# 事务 A
begin;
update user set name = 'jack01' where id = 1;
```
```sql
# 事务 B
begin;
update user set name = 'jack02' where id = 2;
```

#### case2
事务 A 先执行，删除一条不存在的记录，且处于未提交状态；事务 B 会阻塞。在该隔离级别下，事务 A 使用 `next-keylock` 的锁算法，相当于 `gaplock + recordlock` 的组合
```sql
# 事务 A
begin;
delete from user where id = 10;
```
```sql
# 事务 B
begin;
insert into user values(10, 'polobo');
```

```sql
# 事务 A
begin;
delete from user where id = 20;
```
```sql
# 事务 B
begin;
insert into user values(15, 'nojo');
```

```sql
# 事务 A
begin;
delete from user where id = 30;
```
```sql
# 事务 B
begin;
insert into user values(35, 'copo');
```

### 配置隔离级别
```sql
select @@tx_isolation;
```
```sql
show variables like 'transaction_isolation';
show variables like "tx_isolation";
```
```sql
set transaction isolation level read uncommitted;
set session transaction isolation level read uncommitted;
```

### 事务参数
```sql
show global variables like "autocommit";
show session variables like "autocommit";
```
```sql
set session autocommit=0;
```

```sql
# 查找持续时间超过 60s 的事务
select * from information_schema.innodb_trx
where TIME_TO_SEC(timediff(now(), trx_started)) > 60;
```

## 锁
### 悲观锁

### 乐观锁

### 表锁

### 行锁

### 读锁/共享锁
读取操作创建的锁，其他用户可以并发读取数据，但任何事务都不能对数据进行修改（获取数据上的排他锁），直到释放所有共享锁
```sql
SELECT ... LOCK IN SHARE MODE;
```

### 写锁/排他锁
如果事务对数据加上排他锁后，则其他事务不能再对该数据添加任何类型的锁。获准排他锁的事务既能读数据，又能修改数据。对于 `insert`、`update`、`delete`，InnoDB 会自动给涉及的数据加排他锁；对于一般的 `select` 语句，InnoDB 不会加任何锁
```sql
SELECT ... FOR UPDATE;
```


