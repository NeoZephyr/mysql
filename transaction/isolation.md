```sql
DROP TABLE IF EXISTS `heros_temp`;

CREATE TABLE `heros_temp` (
  `id` int(11) NOT NULL,
  `name` varchar(255) NOT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC

INSERT INTO `heros_temp` VALUES (1, '张飞');
INSERT INTO `heros_temp` VALUES (2, '关羽');
INSERT INTO `heros_temp` VALUES (3, '刘备');
```

```sql
SHOW VARIABLES LIKE '%iso%';
```
```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```
```sql
SET autocommit = 0;
```

## 脏读
客户端 2
```sql
begin;
insert into heros_temp values(4, "吕布");
```

客户端 1
```sql
select * from heros_temp;
```


## 不可重复读
客户端 1
```sql
select name from heros_temp where id = 1;
```

客户端 2
```sql
begin;
update heros_temp set name = "张翼德" where id = 1;
```

客户端 1
```sql
select name from heros_temp where id = 1;
```


## 幻读
客户端 1
```sql
select * from heros_temp;
```

客户端 2
```sql
begin;
insert into heros_temp values(4, "吕布");
```

客户端 1
```sql
select * from heros_temp;
```