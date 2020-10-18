```sql
SELECT '王者荣耀' as platform, name FROM heros;
```

DISTINCT 需要放到所有列名的前面
DISTINCT 对后面所有列名的组合进行去重

```sql
SELECT DISTINCT attack_range FROM heros;
```
```sql
SELECT DISTINCT attack_range, name FROM heros;
```

```sql
SELECT name, hp_max FROM heros ORDER BY hp_max DESC;

SELECT name, mp_max, hp_max FROM heros ORDER BY mp_max, hp_max DESC;
SELECT name, mp_max, hp_max FROM heros ORDER BY mp_max DESC, hp_max DESC;
```

```sql
SELECT name, hp_max FROM heros ORDER BY hp_max DESC LIMIT 5;
```


关键字的顺序
```
SELECT
FROM
WHERE
GROUP BY
HAVING
ORDER BY
```

执行顺序
```
FROM > WHERE > GROUP BY > HAVING > SELECT > DISTINCT > ORDER BY > LIMIT
```

在执行这些步骤的时候，每个步骤都会产生一个虚拟表，然后将这个虚拟表传入下一个步骤中作为输入
```sql
SELECT DISTINCT player_id, player_name, count(*) as num   # 5
FROM player JOIN team ON player.team_id = team.team_id    # 1
WHERE height > 1.80                                       # 2
GROUP BY player.team_id                                   # 3
HAVING num > 2                                            # 4
ORDER BY num DESC                                         # 6
LIMIT 2                                                   # 7
```

1. 通过 CROSS JOIN 求笛卡尔积，相当于得到虚拟表 vt1-1
2. 通过 ON 进行筛选，得到虚拟表 vt1-2
3. 添加外部行。如果使用的是左连接、右链接或者全连接，就会涉及到外部行，在虚拟表 vt1-2 的基础上增加外部行，得到虚拟表 vt1-3
4. WHERE 阶段，进行筛选过滤，得到虚拟表 vt2
5. 在虚拟表 vt2 的基础上进行分组（GROUP）和分组过滤（HAVING），得到中间的虚拟表 vt3 和 vt4
6. 在 SELECT 阶段提取想要的字段，然后在 DISTINCT 阶段过滤掉重复的行，得到中间的虚拟表 vt5
7. ORDER BY 阶段，按照指定的字段进行排序，得到虚拟表 vt6
8. LIMIT 阶段，取出指定行的记录，得到虚拟表 vt7


```sql
SELECT name, hp_max FROM heros WHERE hp_max BETWEEN 5399 AND 6811
```
```sql
SELECT name, hp_max, mp_max FROM heros WHERE hp_max > 6000 AND mp_max > 1700 ORDER BY (hp_max + mp_max) DESC
```
```sql
SELECT name, role_main, role_assist, hp_max, mp_max, birthdate
FROM heros 
WHERE (role_main IN ('法师', '射手') OR role_assist IN ('法师', '射手')) 
AND DATE(birthdate) NOT BETWEEN '2016-01-01' AND '2017-01-01'
ORDER BY (hp_max + mp_max) DESC;
```
```sql
SELECT name FROM heros WHERE name LIKE '_%太%';
```
