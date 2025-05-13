### Задание 2.  ускорить запрос "max + left join", добиться времени выполнения < 10ms (запрос со статистикой выполнения)
```
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```

**Решение**

```
EXPLAIN (ANALYZE, BUFFERS)
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```

*Результат*

```
Planning:
  Buffers: shared hit=18 read=6 dirtied=2
Planning Time: 145.117 ms
JIT:
  Functions: 41
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 23.849 ms, Inlining 0.000 ms, Optimization 112.800 ms, Emission 254.137 ms, Total 390.786 ms
Execution Time: 379093.772 ms
```

У нас уже был создан индекс по полю ```t1.id```
Создаём индексы для поля t1.name, это позволяет ускорить фильтрацию по этому полю.

```
CREATE INDEX IF NOT EXISTS idx_t1_name ON t1(name);
```

*Результат*

```
Planning:
  Buffers: shared hit=8
Planning Time: 0.228 ms
JIT:
  Functions: 41
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 2.714 ms, Inlining 0.000 ms, Optimization 1.286 ms, Emission 171.423 ms, Total 175.424 ms
Execution Time: 21300.742 ms
```
Создаём паттерновые индексы, так как обычный индекс не используется для LIKE 'a%' без дополнительных настроек.

```
CREATE INDEX IF NOT EXISTS idx_t1_name_prefix ON t1(name text_pattern_ops);
```

Также можно переписать запрос, чтобы фильтрация происходила до соединения таблиц;

```
EXPLAIN (ANALYZE, BUFFERS)
SELECT max(t2.day)
FROM t2 JOIN ( SELECT id FROM t1 WHERE name LIKE 'a%') filtered_t1 ON t2.t_id = filtered_t1.id;
```

*Результат*

```
Planning:
  Buffers: shared hit=52 read=1
Planning Time: 8.338 ms
JIT:
  Functions: 41
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 2.589 ms, Inlining 0.000 ms, Optimization 1.661 ms, Emission 26.585 ms, Total 30.835 ms
Execution Time: 19651.067 ms
```

```
CREATE INDEX idx_t1_name_a ON t1(id) WHERE name LIKE 'a%';
```

*Результат*

```
Planning:
  Buffers: shared hit=23 read=1
Planning Time: 0.723 ms
JIT:
  Functions: 29
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 2.214 ms, Inlining 0.000 ms, Optimization 1.036 ms, Emission 18.355 ms, Total 21.605 ms
Execution Time: 12888.285 ms
```

В таблице ```t2``` для столбца ```day``` указан тип данных ```text```, при том что данные типа ```date```. При изменении типа данных исходный запрос с удалением ```left``` для соединения выдаёт следующий результат:

Изменение типа данных:

```
alter table t2 alter column day type date using day::date;
```
Запрос:
```
EXPLAIN (ANALYZE, BUFFERS)
select max(t2.day) from t2 inner join t1 on t2.t_id = t1.id and t1.name like 'a%';
```

Результат:

```
Planning:
  Buffers: shared hit=8
Planning Time: 0.301 ms
Execution Time: 4895.988 ms
```

