### Задача 1: Необходимо написать запрос, который позволит понять, идентичны ли данные в двух таблицах. Порядок хранения данных в таблицах значения не имеет.

```sql
CREATE TABLE t1(a INTEGER, b INTEGER); 
CREATE TABLE t2(a INTEGER, b INTEGER);
INSERT INTO t1(a, b) VALUES
(1,1),
(2,2),
(2,2),
(3,3),
(4,4);
INSERT INTO t2(a, b) VALUES
(1,1),
(2,2),
(2,2),
(3,3),
(4,4);

```
Примеры таблиц  
T1   
|a|b|
|---|---|
|1|1|
|2|2|
|2|2|
|3|3|
|4|4|

T2
|a|b|
|---|---|
|1|1|
|2|2|
|2|2|
|3|3|
|4|4|

### Решение 1
```sql
-- Решенеие через множества
SELECT  CASE 
	WHEN EXISTS (SELECT * FROM t1 EXCEPT SELECT * FROM t2) = FALSE  
       	AND  EXISTS (SELECT * FROM t2 EXCEPT SELECT * FROM t1) = FALSE THEN 'EQUAL'
        ELSE 'NOT EQUAL'
        END as eq
-- Если разность А - В = 0 и В - А = 0 то таблицы идентичны
```

### Задача 2: Имеется таблица без первичного ключа. Известно, что в таблице имеется задвоение данных. Необходимо удалить дубликаты из таблицы.
```sql
CREATE TABLE t1(a INTEGER, b INTEGER);
INSERT INTO t1(a, b) VALUES
(1,1),
(2,2),
(2,2),
(3,3);
```
Пример таблицы
T1   
|a|b|
|---|---|
|1|1|
|2|2|
|2|2|
|3|3|

### Решение 2
```sql
--postgresql 
-- Через создание новой таблицы и копирование в неё уникальных записей
CREATE TABLE t2 (like t1);
INSERT INTO t2 (a, b) 
SELECT DISTINCT A, b
FROM t1;
DROP t1;
ALTER table t2
RENAME TO t1;

-- MYSQL
WITH cte_dup AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY a,b) AS RowNum
    FROM t1
)
DELETE FROM cte_dup WHERE RowNum > 1;

```

### Задача 3: Есть таблица с данными в виде дерева. Необходимо написать запрос для получения дерева от корневого узла, узел 5 и все его потомки не должны попасть в результат, нужно вывести для каждого узла имя его родителя, данные отсортировать в порядке возрастания ID с учетом иерархии  
```sql
CREATE TABLE  t1
    (id INTEGER, -- идентификатор узла
    pid INTEGER, -- идентификатор родительского узла
    nam VARCHAR(255)); -- наименование

INSERT INTO t1 (id, pid, nam) VALUES
(1, 	NULL,  'Корень'),
(2, 	1, 	'Узел2'),
(3, 	1, 	'Узел3'),
(4, 	2, 	'Узел4'),
(5, 	4, 	'Узел5'),
(6,	5, 	'Узел6'),
(7, 	4, 	'Узел7');
```

Пример данных:  
ID |PID|NAM|
---|---|---|
1	|	|Корень|
2	|1 	|Узел2|
3	|1	|Узел3
4	|2	|Узел4
5	|4	|Узел5
6	|5	|Узел6
7	|4	|Узел7

Требуемый результат:  
ID	|PID	|NAM	|PARENT_NAM|
---|---|---|---|
1|		|Корень|	
2|	1|	Узел2|	Корень
4|	2|	Узел4|	Узел2
7|	4|	Узел7|	Узел4
3|	1|	Узел3|	Корень

### Решение 3
```sql
-- Postgresql not recursive (not same order)
SELECT t.id, t.pid, t.nam, t2.nam parent_nam 
FROM t 
LEFT JOIN t as t2 ON t.pid = t2.id 
WHERE t.pid IS NULL OR t.pid <> 5 and t.id <> 5

-- Postgresql recursive (not same order)

WITH RECURSIVE r  AS (
SELECT t.id, t.pid, t.nam, NULL::VARCHAR(255) as parent_nam 
FROM t 
WHERE t.pid IS NULL  
UNION 
SELECT t.id, t.pid, t.nam, r.nam as parent_nam 
FROM t
JOIN r on t.pid = r.id 
WHERE t.id != 5 AND t.pid != 5
)

SELECT * FROM r ;
```

### Задача 4: Имеется таблица курсов валют. Курс валюты устанавливается не на каждую календарную дату и действует до следующей смены курса. Уникальный ключ: curr_id + date_rate. Напишите запрос, который покажет действующее значение курса заданной валюты на любую заданную календарную дату.  
```sql
CREATE TABLE rates(curr_id INTEGER, -- ид валюты
                   date_rate DATE, -- дата курса
                   rate INTEGER); -- значение курса
INSERT INTO rates (curr_id, date_rate, rate) VALUES
(1, '2010.01.01', 30),
(2, '2010.01.01', 40),
(1, '2010.01.02', 32),
(1, '2010.01.05', 33),
(2, '2010.01.10', 41),
(2, '2010.01.15', 42);
```

Пример данных:  
CURR_ID|	DATE_RATE|	RATE|
|---|---|---|
1|		01.01.2010|	30	
2|		01.01.2010|	40	
1|		02.01.2010|	32	
1|		05.01.2010|	33	
2|		10.01.2010|	41	
2|		15.01.2010|	42	

Требуемый результат:  
Для валюты 1 на 03.01.2010 получить курс 32  
Для валюты 2 на 10.01.2010 получить курс 41

### Решение 4
```sql
CREATE OR REPLACE FUNCTION rateDates (dates DATE, currId INT) RETURNS TABLE (rate INT) AS $$
BEGIN
  RETURN QUERY (
    SELECT rates.rate 
    FROM rates
	WHERE curr_id = currId AND date_rate in (
	SELECT max(date_rate)
  	FROM rates
  	WHERE date_rate <= dates
  	LIMIT 1
)       
  );
END;
$$ LANGUAGE plpgsql;

SELECT * FROM rateDates('2010-01-03', 1) -- 32  
SELECT * FROM rateDates('2010-01-10', 2) -- 41
```

### Задача 5: Имеется таблица с данными по платежным документам. Необходимо написать запрос, который выведет данные общей суммой за каждый месяц по типам документов с итогами по каждому типу и общим итогом.
```sql
CREATE TABLE payments
  (id INTEGER,
  paytype INTEGER,
  paydate DATE,
  paysum INTEGER);
INSERT INTO t1 (id, paytype, paydate, paysum) VALUES 
(1, 	1, 	'2012.01.01.', 	100),
(2 ,	1, 	'2012.01.02', 	200),
(3 ,    1 ,     '2012.01.03', 	300),
(4 ,	1 ,	'2012.02.01', 	400),
(5 ,	1, 	'2012.02.01',	500),
(6 ,    2,	'2012.01.01' ,	600),
(7 ,	2,	'2012.02.01' ,	700),
(8 ,	2 ,	'2012.01.04',	800),
(9 ,	2 ,	'2012.05.01' ,	900),
(10 ,	2 ,	'2012.06.01' ,	1000),
(11 ,	3 ,	'2012.01.10' ,	1100),
(12 ,	3 ,	'2012.03.01',	1200),
(13 ,	3 ,	'2012.05.01' ,	1300),
(14 ,	3 ,	'2012.05.05',	1400),
(15 ,	3, 	'2012.06.01',	1500);
```
Пример данных:
  ID|	PAY_TYPE|	PAY_DATE|	PAY_SUM|
  ---|---|---|---|
1|	1 |		01.01.2012|	100
2|	1	|	02.01.2012|	200
3|	1	|	03.01.2012|	300
4|	1	|	01.02.2012|	400
5|	1	|	01.02.2012|	500
6|	2	|	01.01.2012|	600
7|	2	|	01.02.2012|	700
8|	2	|	01.04.2012|	800
9|	2	|	01.05.2012|	900
10|	2	|	01.06.2012|	1000
11|	3	|	10.01.2012|	1100
12|	3	|	01.03.2012|	1200
13|	3	|	01.05.2012|	1300
14|	3	|	05.05.2012|	1400
15|	3	|	01.06.2012|	1500

Требуемый результат:  
PAY_TYPE|	MON|	SM|
---|---|---|
1|		01.2012|	600
1|		02.2012|	900
1	|	NULL		|1500|
2	|	01.2012	|600
2	|	02.2012	|700
2	|	04.2012	|800
2	|	05.2012	|900
2	|	06.2012	|1000
2	|		NULL	|4000
3	|	01.2012|	1100
3	|	03.2012|	1200
3	|	05.2012	|2700
3	|	06.2012	|1500
3	|		NULL	|6500
NULL	|		NULL	|12000

 ### Решение 5
```sql
SELECT paytype, 
       monthyear AS mon,
       SUM(paysum) AS sm
FROM (
    SELECT paytype,
           to_char(paydate, 'mm.yyyy') AS monthyear,
           paysum
    FROM t1) t2 
GROUP BY ROLLUP(paytype, monthyear)
ORDER BY 1, 2;
```

### Задача 6: Дана таблица валют (справочник), необходимо написать запрос, который возвращает отсортированный список валют в алфавитном порядке по столбцу ISO_CODE, причем первыми должны идти основные валюты, с которыми работает банк: RUR, USD, EUR.  
```sql
CREATE TABLE t
  (iso_code VARCHAR(255),
  iso_name VARCHAR(255));
INSERT INTO t (iso_code, iso_name) VALUES
('USD', 'Доллар США'),
('CHF', 'Швейцарский франк'),
('AED', 'Дирхам (ОАЭ)'),
('RUR', 'Российский Рубль'),
('ETB', 'Эфиопский быр'),
('MXN', 'Мексиканское песо'),
('EUR', 'ЕВРО');
```
Пример данных:  
ISO_CODE|	ISO_NAME|
---|---|
USD |		Доллар США|
CHF	|	Швейцарский франк|
AED	|	Дирхам (ОАЭ)|
RUR	|	Российский Рубль|
ETB	|	Эфиопский быр|
MXN	|	Мексиканское песо|
EUR	|	ЕВРО|

Требуемый результат:  
ISO_CODE|	ISO_NAME|
---|---|
RUR |	Российский Рубль|
USD	|	Доллар США|
EUR	|	ЕВРО|
AED	|	Дирхам (ОАЭ)|
CHF	|	Швейцарский франк|
ETB	|	Эфиопский быр|
MXN	|	Мексиканское песо|

### Решение 6
```sql
SELECT iso_code, iso_name
FROM
(SELECT  *,
	CASE 
	WHEN iso_code = 'RUR' THEN 1
        WHEN iso_code = 'USD' THEN 2
        WHEN iso_code = 'EUR' THEN 3
        ELSE 42
	END as rank_
FROM t) tab 
ORDER BY rank_, iso_code
```
