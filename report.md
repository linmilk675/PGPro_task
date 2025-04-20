## Оптимизация SQL-запросов 

Используется postgreSQL 16.7.0

### Задание 0. - выполнить скрипт генерации БД

```
-- turn off parallel queries
alter system set max_parallel_workers_per_gather=0;
-- apply changes
select pg_reload_conf();

-- t1 table has 10e7 rows
create table if not exists t1 as
 select a.id as id
      , cast(a.id - floor( random() *(a.id) ) as integer ) as parent_id
      , substr( md5( random()::text ), 0, 30 )  as name
 from generate_series ( 1, 10000000 ) a(id);

-- t2 table has 5*10e6 rows
create table if not exists t2  as
select row_number() over() as id
     , id as t_id
     , to_char(date_trunc('day', now()- random()*'1 year'::interval),'yyyymmdd') as day
from t1
order by random() 
limit 5000000;

-- initial database state: only tables t1 and t2
--
--# \dti+
--                                       List of relations
-- Schema | Name | Type  |  Owner   | Table | Persistence | Access method |  Size  | Description 
----------+------+-------+----------+-------+-------------+---------------+--------+-------------
-- public | t1   | table | postgres |       | permanent   | heap          | 731 MB | 
-- public | t2   | table | postgres |       | permanent   | heap          | 249 MB | 
--(2 rows)
```

**Заметки:**
При создании таблицы t2 можно использовать TABLESAMPLE SYSTEM, он (метод) работает быстрее, чем ORDER BY RANDOM

### Задание 1. ускорить простой запроc, добиться времени выполнения < 10ms (запрос со статистикой выполнения)
```
select name from t1 where id = 50000;
```

**Решение.**
Когда выполняется ```WHERE id = 50000``` делается полный скан таблицы, а при 10 000 000 строк это занимает очень много времени.

```
EXPLAIN (ANALYZE, BUFFERS)
SELECT name FROM t1 WHERE id = 50000;
```

*Результат*

Для оптимизации можно использовать индексирование. Они работают как указатель, который направляет СУБД к нужным строкам вместо того, чтобы сканировать всю таблицу.

*Создание индекса*

```
CREATE INDEX IF NOT EXISTS idx_t1_id ON t1(id);
```

*Выполенение запроса после добавления индекса*

```
EXPLAIN (ANALYZE, BUFFERS)
SELECT name FROM t1 WHERE id = 50000;
```

*Результат*

```

```

### Задание 2.  ускорить запрос "max + left join", добиться времени выполнения < 10ms (запрос со статистикой выполнения)
```
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```

**Решение**

Фильтрация происходит на этапе соединения, т.е. это условие не отбрасывает строки из t2 => выполняется полный скан таблиц.
Также выполняется последовательное сканирование

```
EXPLAIN (ANALYZE, BUFFERS)
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```

*Результат*
```
```

Создаём опять индексы для поля t1.id, это позволяет ускорить соединение по этому полю.
```
CREATE INDEX IF NOT EXISTS idx_t1_id ON t1(id);
```
Создаём паттерновые индексы, так как обычный индекс не используется для LIKE 'a%' без дополнительных настроек.
```
CREATE INDEX IF NOT EXISTS idx_t1_name_prefix ON t1(name text_pattern_ops);
```
Обновляем статистику
```
ANALYZE t1;
ANALYZE t2;
```


**Выполенение запроса после преобразований**
```
EXPLAIN (ANALYZE, BUFFERS)
SELECT max(t2.day)
FROM t2
LEFT JOIN t1 ON t2.t_id = t1.id AND t1.name LIKE 'a%';
```

*Результат*
```
```
