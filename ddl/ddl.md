## database
```sql
CREATE DATABASE nba;
DROP DATABASE nba;
```


## table
```sql
DROP TABLE IF EXISTS `player`;

CREATE TABLE `player` (
  `player_id` int(11) NOT NULL AUTO_INCREMENT,
  `team_id` int(11) NOT NULL,
  `player_name` varchar(255) NOT NULL,
  `height` float(3,2) DEFAULT '0.00',
  PRIMARY KEY (`player_id`) USING BTREE,
  UNIQUE KEY `player_name` (`player_name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC
```

添加字段
```sql
ALTER TABLE player ADD (age int(11));
```

修改字段名
```sql
ALTER TABLE player CHANGE age player_age int(11) NULL;
```

修改字段的数据类型
```sql
ALTER TABLE player MODIFY column player_age float(3,1)
```

删除字段
```sql
ALTER TABLE player DROP COLUMN player_age;
```


