## 笛卡尔积
```sql
SELECT * FROM player, team;
SELECT * FROM player CROSS JOIN team;
```
笛卡尔积也称为交叉连接，也就是是 CROSS JOIN，它的作用就是可以把任意表进行连接


## 等值连接
用两张表中都存在的列进行连接

```sql
SELECT player_id, a.team_id, player_name, height, team_name FROM player AS a, team AS b WHERE a.team_id = b.team_id;
```
```sql
SELECT player_id, a.team_id, player_name, height, team_name FROM player AS a JOIN team AS b ON a.team_id = b.team_id;
```
```sql
SELECT player_id, team_id, player_name, height, team_name FROM player JOIN team USING(team_id);
```


## 非等值连接
```sql
DROP TABLE IF EXISTS `height_grades`;

CREATE TABLE `height_grades` (
  `height_level` varchar(255) NOT NULL COMMENT '身高等级',
  `height_lowest` float(3,2) NOT NULL COMMENT '该等级范围中的最低身高',
  `height_highest` float(3,2) NOT NULL COMMENT '该等级范围中的最高身高'
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC
```
```sql
INSERT INTO `height_grades` VALUES ('A', 2.00, 2.50);
INSERT INTO `height_grades` VALUES ('B', 1.90, 1.99);
INSERT INTO `height_grades` VALUES ('C', 1.80, 1.89);
INSERT INTO `height_grades` VALUES ('D', 1.60, 1.79);
```

```sql
SELECT p.player_name, p.height, h.height_level
FROM player AS p, height_grades AS h
WHERE p.height BETWEEN h.height_lowest AND h.height_highest;
```
```sql
SELECT p.player_name, p.height, h.height_level
FROM player as p JOIN height_grades as h
ON height BETWEEN h.height_lowest AND h.height_highest;
```


## 外连接
左外连接
```sql
SELECT * FROM player LEFT JOIN team on player.team_id = team.team_id;
```

右外连接
```sql
SELECT * FROM player RIGHT JOIN team on player.team_id = team.team_id;
```

全外连接
```sql
SELECT * FROM player FULL JOIN team ON player.team_id = team.team_id;
```

## 自连接
```sql
SELECT b.player_name, b.height FROM player as a , player as b WHERE a.player_name = '布雷克-格里芬' and a.height < b.height;
```

```sql
SELECT b.player_name, b.height FROM player as a JOIN player as b ON a.player_name = '布雷克-格里芬' and a.height < b.height;
```

```sql
SELECT player_name, height FROM player WHERE height > (SELECT height FROM player WHERE player_name = '布雷克-格里芬');
```
