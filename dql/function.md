## 算术函数
```sql
SELECT ABS(-2);
SELECT MOD(101, 6);
SELECT ROUND(37.25, 1);
```

```sql
SELECT name, ROUND(attack_growth, 1) FROM heros;
```
```sql
SELECT name, hp_max FROM heros WHERE hp_max = (SELECT MAX(hp_max) FROM heros);
```


## 字符串函数
```sql
SELECT CONCAT('abc', 123);
SELECT LENGTH('你好✨');
SELECT CHAR_LENGTH('你好✨');
SELECT LOWER('ABC');
SELECT UPPER('abc');
SELECT REPLACE('fabcd', 'abc', 123);
SELECT SUBSTRING('fabcd', 1, 3);
```


## 日期函数
```sql
SELECT CURRENT_DATE();
SELECT CURRENT_TIME();
SELECT CURRENT_TIMESTAMP();
SELECT EXTRACT(YEAR FROM '2019-04-03');
SELECT DATE('2019-04-01 12:00:05');
SELECT TIME('2019-04-01 12:00:05');
```

```sql
SELECT name, EXTRACT(YEAR FROM birthdate) AS birthdate FROM heros WHERE birthdate is NOT NULL;
SELECT name, YEAR(birthdate) AS birthdate FROM heros WHERE birthdate is NOT NULL;
```
```sql
SELECT name, birthdate FROM heros WHERE DATE(birthdate) > '2016-10-01';
```

```sql
SELECT name, birthdate FROM heros WHERE birthdate > '2016-10-01';
```


## 转换函数
```sql
SELECT CAST(3.9999 AS DECIMAL(8, 2));
SELECT COALESCE(null, null, 2);
```


## 聚集函数
```sql
SELECT COUNT(*) FROM heros WHERE hp_max > 6000;

SELECT COUNT(role_assist) FROM heros WHERE hp_max > 6000;
```
```sql
SELECT COUNT(*), AVG(hp_max), MAX(mp_max), MIN(attack_max), SUM(defense_max) FROM heros WHERE role_main = '射手' or role_assist = '射手';
```
```sql
SELECT MIN(CONVERT(name USING gbk)), MAX(CONVERT(name USING gbk)) FROM heros;
SELECT MIN(name), MAX(name) FROM heros;
```
```sql
SELECT ROUND(AVG(DISTINCT hp_max), 2) FROM heros;
```

```sql
SELECT COUNT(*), role_main FROM heros GROUP BY role_main;
```
```sql
SELECT COUNT(*) as num, role_main, role_assist FROM heros GROUP BY role_main, role_assist ORDER BY num DESC;
```
```sql
SELECT COUNT(*) as num, role_main, role_assist FROM heros GROUP BY role_main, role_assist HAVING num > 5 ORDER BY num DESC;
```

