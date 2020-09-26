## 非关联子查询
子查询从数据表中查询了数据结果，如果这个数据结果只执行一次，然后这个数据结果作为主查询的条件进行执行，这样的子查询叫做非关联子查询


## 关联子查询
如果子查询需要执行多次，即采用循环的方式，先从外部查询开始，每次都传入子查询进行查询，然后再将结果反馈给外部，这种嵌套的执行方式就称为关联子查询

```sql
DROP TABLE IF EXISTS `player`;

CREATE TABLE `player` (
  `player_id` int(11) NOT NULL AUTO_INCREMENT COMMENT '球员ID',
  `team_id` int(11) NOT NULL COMMENT '球队ID',
  `player_name` varchar(255) NOT NULL COMMENT '球员姓名',
  `height` float(3,2) DEFAULT NULL COMMENT '球员身高',
  PRIMARY KEY (`player_id`) USING BTREE,
  UNIQUE KEY `player_name` (`player_name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC
```

```sql
INSERT INTO `player` VALUES (10001, 1001, '韦恩-艾灵顿', 1.93);
INSERT INTO `player` VALUES (10002, 1001, '雷吉-杰克逊', 1.91);
INSERT INTO `player` VALUES (10003, 1001, '安德烈-德拉蒙德', 2.11);
INSERT INTO `player` VALUES (10004, 1001, '索恩-马克', 2.16);
INSERT INTO `player` VALUES (10005, 1001, '布鲁斯-布朗', 1.96);
INSERT INTO `player` VALUES (10006, 1001, '兰斯顿-加洛韦', 1.88);
INSERT INTO `player` VALUES (10007, 1001, '格伦-罗宾逊三世', 1.98);
INSERT INTO `player` VALUES (10008, 1001, '伊斯梅尔-史密斯', 1.83);
INSERT INTO `player` VALUES (10009, 1001, '扎扎-帕楚里亚', 2.11);
INSERT INTO `player` VALUES (10010, 1001, '乔恩-洛伊尔', 2.08);
INSERT INTO `player` VALUES (10011, 1001, '布雷克-格里芬', 2.08);
INSERT INTO `player` VALUES (10012, 1001, '雷吉-巴洛克', 2.01);
INSERT INTO `player` VALUES (10013, 1001, '卢克-肯纳德', 1.96);
INSERT INTO `player` VALUES (10014, 1001, '斯坦利-约翰逊', 2.01);
INSERT INTO `player` VALUES (10015, 1001, '亨利-埃伦森', 2.11);
INSERT INTO `player` VALUES (10016, 1001, '凯里-托马斯', 1.91);
INSERT INTO `player` VALUES (10017, 1001, '何塞-卡尔德隆', 1.91);
INSERT INTO `player` VALUES (10018, 1001, '斯维亚托斯拉夫-米凯卢克', 2.03);
INSERT INTO `player` VALUES (10019, 1001, '扎克-洛夫顿', 1.93);
INSERT INTO `player` VALUES (10020, 1001, '卡林-卢卡斯', 1.85);
INSERT INTO `player` VALUES (10021, 1002, '维克多-奥拉迪波', 1.93);
INSERT INTO `player` VALUES (10022, 1002, '博扬-博格达诺维奇', 2.03);
INSERT INTO `player` VALUES (10023, 1002, '多曼塔斯-萨博尼斯', 2.11);
INSERT INTO `player` VALUES (10024, 1002, '迈尔斯-特纳', 2.11);
INSERT INTO `player` VALUES (10025, 1002, '赛迪斯-杨', 2.03);
INSERT INTO `player` VALUES (10026, 1002, '达伦-科里森', 1.83);
INSERT INTO `player` VALUES (10027, 1002, '韦斯利-马修斯', 1.96);
INSERT INTO `player` VALUES (10028, 1002, '泰瑞克-埃文斯', 1.98);
INSERT INTO `player` VALUES (10029, 1002, '道格-迈克德莫特', 2.03);
INSERT INTO `player` VALUES (10030, 1002, '科里-约瑟夫', 1.91);
INSERT INTO `player` VALUES (10031, 1002, '阿龙-霍勒迪', 1.85);
INSERT INTO `player` VALUES (10032, 1002, 'TJ-利夫', 2.08);
INSERT INTO `player` VALUES (10033, 1002, '凯尔-奥奎因', 2.08);
INSERT INTO `player` VALUES (10034, 1002, '埃德蒙-萨姆纳', 1.96);
INSERT INTO `player` VALUES (10035, 1002, '达文-里德', 1.98);
INSERT INTO `player` VALUES (10036, 1002, '阿利兹-约翰逊', 2.06);
INSERT INTO `player` VALUES (10037, 1002, '伊凯·阿尼博古', 2.08);
```

```sql
DROP TABLE IF EXISTS `team`;

CREATE TABLE `team` (
  `team_id` int(11) NOT NULL COMMENT '球队ID',
  `team_name` varchar(255) NOT NULL COMMENT '球队名称',
  PRIMARY KEY (`team_id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC
```

```sql
INSERT INTO `team` VALUES (1001, '底特律活塞');
INSERT INTO `team` VALUES (1002, '印第安纳步行者');
INSERT INTO `team` VALUES (1003, '亚特兰大老鹰');
```

```sql
DROP TABLE IF EXISTS `team_score`;

CREATE TABLE `team_score` (
  `game_id` int(11) NOT NULL COMMENT '比赛ID',
  `h_team_id` int(11) NOT NULL COMMENT '主队ID',
  `v_team_id` int(11) NOT NULL COMMENT '客队ID',
  `h_team_score` int(11) NOT NULL COMMENT '主队得分',
  `v_team_score` int(11) NOT NULL COMMENT '客队得分',
  `game_date` date DEFAULT NULL COMMENT '比赛时间',
  PRIMARY KEY (`game_id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC
```

```sql
INSERT INTO `team_score` VALUES (10001, 1001, 1002, 102, 111, '2019-04-01');
INSERT INTO `team_score` VALUES (10002, 1002, 1003, 135, 134, '2019-04-10');
```

```sql
DROP TABLE IF EXISTS `player_score`;

CREATE TABLE `player_score` (
  `game_id` int(11) NOT NULL COMMENT '比赛ID',
  `player_id` int(11) NOT NULL COMMENT '球员ID',
  `is_first` tinyint(1) NOT NULL COMMENT '是否首发',
  `playing_time` int(11) NOT NULL COMMENT '该球员本次比赛出场时间',
  `rebound` int(11) NOT NULL COMMENT '篮板球',
  `rebound_o` int(11) NOT NULL COMMENT '前场篮板',
  `rebound_d` int(11) NOT NULL COMMENT '后场篮板',
  `assist` int(11) NOT NULL COMMENT '助攻',
  `score` int(11) NOT NULL COMMENT '比分',
  `steal` int(11) NOT NULL COMMENT '抢断',
  `blockshot` int(11) NOT NULL COMMENT '盖帽',
  `fault` int(11) NOT NULL COMMENT '失误',
  `foul` int(11) NOT NULL COMMENT '犯规',
  `shoot_attempts` int(11) NOT NULL COMMENT '总出手',
  `shoot_hits` int(11) NOT NULL COMMENT '命中',
  `shoot_3_attempts` int(11) NOT NULL COMMENT '3分出手',
  `shoot_3_hits` int(11) NOT NULL COMMENT '3分命中',
  `shoot_p_attempts` int(11) NOT NULL COMMENT '罚球出手',
  `shoot_p_hits` int(11) NOT NULL COMMENT '罚球命中'
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC
```

```sql
INSERT INTO `player_score` VALUES (10001, 10001, 1, 38, 4, 1, 3, 2, 26, 0, 1, 0, 3, 19, 10, 13, 4, 4, 2);
INSERT INTO `player_score` VALUES (10001, 10002, 1, 30, 6, 4, 2, 4, 22, 0, 0, 6, 3, 19, 8, 5, 1, 5, 5);
INSERT INTO `player_score` VALUES (10001, 10003, 1, 37, 17, 7, 10, 5, 18, 4, 0, 3, 4, 18, 8, 1, 0, 5, 2);
INSERT INTO `player_score` VALUES (10001, 10004, 1, 42, 6, 1, 5, 2, 14, 0, 4, 1, 2, 10, 4, 7, 4, 2, 2);
INSERT INTO `player_score` VALUES (10001, 10005, 1, 19, 2, 0, 2, 2, 0, 2, 0, 1, 1, 1, 0, 1, 0, 0, 0);
INSERT INTO `player_score` VALUES (10001, 10006, 0, 23, 2, 2, 0, 1, 9, 1, 0, 0, 2, 10, 3, 3, 2, 1, 1);
INSERT INTO `player_score` VALUES (10001, 10007, 0, 13, 1, 1, 0, 1, 7, 0, 0, 0, 2, 4, 2, 2, 1, 2, 2);
INSERT INTO `player_score` VALUES (10001, 10008, 0, 20, 2, 0, 2, 3, 6, 0, 0, 3, 3, 5, 3, 0, 0, 0, 0);
INSERT INTO `player_score` VALUES (10001, 10009, 0, 11, 1, 0, 1, 1, 0, 0, 0, 1, 4, 0, 0, 0, 0, 0, 0);
INSERT INTO `player_score` VALUES (10001, 10010, 0, 7, 0, 0, 0, 0, 0, 0, 0, 0, 0, 2, 0, 1, 0, 0, 0);
INSERT INTO `player_score` VALUES (10002, 10022, 1, 37, 7, 1, 6, 6, 19, 3, 0, 1, 3, 16, 7, 3, 1, 4, 4);
INSERT INTO `player_score` VALUES (10002, 10025, 1, 34, 9, 1, 8, 5, 19, 0, 0, 5, 1, 12, 8, 0, 0, 4, 3);
INSERT INTO `player_score` VALUES (10002, 10024, 1, 34, 6, 0, 6, 0, 17, 3, 5, 0, 2, 7, 5, 3, 2, 6, 5);
INSERT INTO `player_score` VALUES (10002, 10028, 1, 27, 3, 0, 3, 3, 13, 1, 1, 3, 1, 10, 4, 6, 4, 2, 1);
INSERT INTO `player_score` VALUES (10002, 10030, 1, 31, 1, 0, 1, 3, 4, 2, 0, 1, 2, 9, 2, 3, 0, 0, 0);
INSERT INTO `player_score` VALUES (10002, 10023, 0, 23, 12, 4, 8, 3, 18, 0, 0, 3, 6, 10, 8, 0, 0, 2, 2);
INSERT INTO `player_score` VALUES (10002, 10029, 0, 24, 2, 1, 1, 2, 11, 0, 0, 1, 2, 8, 5, 3, 1, 0, 0);
INSERT INTO `player_score` VALUES (10002, 10031, 0, 25, 1, 0, 1, 5, 10, 0, 1, 2, 3, 4, 3, 3, 2, 4, 2);
INSERT INTO `player_score` VALUES (10002, 10032, 0, 4, 1, 0, 1, 0, 0, 0, 1, 1, 0, 1, 0, 1, 0, 0, 0);
```


非关联子查询
```sql
SELECT player_name, height FROM player WHERE height = (SELECT max(height) FROM player);
```

关联子查询
```sql
SELECT player_name, height, team_id FROM player AS a WHERE height > (SELECT avg(height) FROM player AS b WHERE a.team_id = b.team_id);
```

```sql
SELECT player_id, team_id, player_name FROM player WHERE EXISTS (SELECT player_id FROM player_score WHERE player.player_id = player_score.player_id)
```
```sql
SELECT player_id, team_id, player_name FROM player WHERE NOT EXISTS (SELECT player_id FROM player_score WHERE player.player_id = player_score.player_id)
```

```sql
SELECT player_id, team_id, player_name FROM player WHERE player_id in (SELECT player_id FROM player_score WHERE player.player_id = player_score.player_id);
```


EXISTS 的实现，相当于外表循环，逻辑类似于：
```
for i in A
    for j in B
        if j.cc == i.cc then
```
```sql
SELECT * FROM A WHERE EXIST (SELECT cc FROM B WHERE B.cc = A.cc);
```

IN 的实现的逻辑类似于：
```
for i in B
    for j in A
        if j.cc == i.cc then
```
```sql
SELECT * FROM A WHERE cc IN (SELECT cc FROM B);
```

A 表有 n 条数据，B 表有 m 条数据
用 IN: m * log (n)
用 EXISTS: n * log (m)

A 表小就用 EXISTS，B 表小就用 IN


```sql
SELECT player_id, player_name, height FROM player WHERE height > ANY (SELECT height FROM player WHERE team_id = 1002);
```

```sql
SELECT player_id, player_name, height FROM player WHERE height > ALL (SELECT height FROM player WHERE team_id = 1002);
```

```sql
SELECT team_name, (SELECT count(*) FROM player WHERE player.team_id = team.team_id) AS player_num FROM team;
```
