事务支持是在引擎层实现的，有的存储引擎支持事务，有的存储引擎不支持事务。事务具有原子性、一致性、隔离性、持久性

## 启动方式
显式启动事务语句
```sql
-- 开启事务
START TRANSACTION;
```
```sql
-- 开启事务
BEGIN;
```

```sql
-- 提交事务
COMMIT;
```

```sql
-- 回滚
ROLLBACK;
```

关闭自动提交
```sql
set autocommit = 0
```
将这个线程的自动提交关掉。意味着只执行一个 select 语句，这个事务就启动了，而且并不会自动提交。这个事务持续存在直到你主动执行 commit 或 rollback 语句，或者断开连接。建议总是使用 set autocommit=1, 通过显式语句的方式来启动事务


通过 SET MAX_EXECUTION_TIME 命令，来控制每个语句执行的最长时间，避免单个语句意外执行太长时间
