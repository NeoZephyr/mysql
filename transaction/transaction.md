## 事务
事务支持是在引擎层实现的，有的存储引擎支持事务，有的存储引擎不支持事务。事务具有原子性、一致性、隔离性、持久性

### 自动提交
```sql
# 关闭自动提交
set autocommit = 0;
```
```sql
# 开启自动提交
set autocommit = 1;
```


## 控制语句
### 开启事务
```sql
START TRANSACTION;
```
```sql
BEGIN;
```

### 提交事务
```sql
COMMIT;
```

当提交事务后，对数据库的修改是永久性的

### 回滚
```sql
ROLLBACK;
```
```sql
ROLLBACK TO [SAVEPOINT];
```

撤销正在进行的所有没有提交的修改，或者将事务回滚到某个保存点

### 创建保存点
```sql
SAVEPOINT;
```

一个事务中可以存在多个保存点

### 删除某个保存点
```sql
RELEASE SAVEPOINT;
```


```sql
CREATE TABLE test(name varchar(255), PRIMARY KEY (name)) ENGINE=InnoDB;

BEGIN;
INSERT INTO test SELECT '关羽';
COMMIT;

BEGIN;
INSERT INTO test SELECT '张飞';
ROLLBACK;

SELECT * FROM test;
```
