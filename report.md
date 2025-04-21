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
```
  ->  Parallel Seq Scan on t1  (cost=0.00..135424.25 rows=1 width=30) (actual time=117374.837..175395.957 rows=0 loops=3)
        Filter: (id = 50000)
        Rows Removed by Filter: 3333333
        Buffers: shared hit=2113 read=81225
Planning:
  Buffers: shared hit=71 dirtied=1
Planning Time: 0.665 ms
  Timing: Generation 1.746 ms, Inlining 0.000 ms, Optimization 8.656 ms, Emission 75.327 ms, Total 85.728 ms
Execution Time: 175920.709 ms
```

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
Index Scan using idx_t1_id on t1  (cost=0.43..8.45 rows=1 width=30) (actual time=86.162..86.165 rows=1 loops=1)
  Index Cond: (id = 50000)
  Buffers: shared read=4
Planning:
  Buffers: shared hit=19 read=1 dirtied=2
Planning Time: 1.751 ms
Execution Time: 86.813 ms
```

По умолчанию в PostgreSQL задано, что доступ по индексу = дорого (в сравнении с последовательным чтением). Можно снизить random_page_cost:
```
SET random_page_cost = 0.5;
```

**Результат**

```
Index Scan using idx_t1_id on t1  (cost=0.43..1.45 rows=1 width=30) (actual time=0.021..0.022 rows=1 loops=1)
  Index Cond: (id = 50000)
  Buffers: shared hit=4
Planning Time: 0.062 ms
Execution Time: 0.037 ms
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
Planning:
  Buffers: shared hit=79 read=3
Planning Time: 60.265 ms
JIT:
  Functions: 41
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 23.972 ms, Inlining 0.000 ms, Optimization 87.510 ms, Emission 305.140 ms, Total 416.621 ms
Execution Time: 253074.684 ms
```

**Создаём индексы:**

Создаём индексы для поля t1.name, это позволяет ускорить фильтрацию по этому полю.
```
CREATE INDEX IF NOT EXISTS idx_t1_name ON t1(name);
```
Создаём индексы для полей t1.id и t2.id, это позволяет ускорить соединение по этим полям.
```
CREATE INDEX IF NOT EXISTS idx_t1_id ON t1(id);

CREATE INDEX IF NOT EXISTS idx_t2_t_id ON t2(t_id);
```
Создаём паттерновые индексы, так как обычный индекс не используется для LIKE 'a%' без дополнительных настроек.
```
CREATE INDEX IF NOT EXISTS idx_t1_name_prefix ON t1(name text_pattern_ops);
```
Также можно переписать запрос, чтобы фильтрация происходила до соединения таблиц; И поменять left join на inner join, так как нам нужен max(day) только по совпавшим строкам
```
EXPLAIN (ANALYZE, BUFFERS)
SELECT max(t2.day)
FROM t2 JOIN ( SELECT id FROM t1 WHERE name LIKE 'a%') filtered_t1 ON t2.t_id = filtered_t1.id;
```

**Выполенение запроса после преобразований**
```
EXPLAIN (ANALYZE, BUFFERS)
SELECT max(t2.day)
FROM t2 JOIN ( SELECT id FROM t1 WHERE name LIKE 'a%') filtered_t1 ON t2.t_id = filtered_t1.id;
```

*Результат*
```
Planning:
  Buffers: shared hit=16
Planning Time: 0.296 ms
Execution Time: 235108.644 ms
```
