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
Planning Time: 0.182 ms
Execution Time: 5123.452 ms
```

У нас уже был создан индекс по полю ```t1.id```
Создаём индексы для поля t1.name, это позволяет ускорить фильтрацию по этому полю.

```
CREATE INDEX IF NOT EXISTS idx_t1_name ON t1(name);
```

*Результат*

```
Planning:
  Buffers: shared hit=26 read=6
Planning Time: 0.551 ms
Execution Time: 4911.344 ms
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
  Buffers: shared hit=3 read=13
Planning Time: 0.306 ms
Execution Time: 2992.257 ms
```

```
CREATE INDEX IF NOT EXISTS idx_t1_name_a ON t1(id) WHERE name LIKE 'a%';
```

*Результат*

```
Planning:
  Buffers: shared hit=21 read=13
Planning Time: 0.540 ms
Execution Time: 2117.358 ms
```

В таблице ```t2``` для столбца ```day``` указан тип данных ```text```, при том что данные типа ```date```. При изменении типа данных запрос выдаёт следующий результат:

Изменение типа данных:

```
alter table t2 alter column day type date using day::date;
```
Запрос:
```
EXPLAIN (ANALYZE, BUFFERS)
SELECT max(t2.day)
FROM t2 JOIN ( SELECT id FROM t1 WHERE name LIKE 'a%') filtered_t1 ON t2.t_id = filtered_t1.id;
```

Результат:

```
Planning:
  Buffers: shared hit=16
Planning Time: 0.197 ms
Execution Time: 1797.566 ms
```

### Задание 3. ускорить запрос "anti-join", добиться времени выполнения < 10sec (запрос со статистикой выполнения)
```
select day from t2 where t_id not in ( select t1.id from t1 );
```

**Решение**

Создаём индексы

```
CREATE INDEX IF NOT EXISTS idx_t1_id ON t1(id);
CREATE INDEX IF NOT EXISTS idx_t2_id ON t2(t_id);
```
Меняем запрос
```
EXPLAIN (ANALYZE, BUFFERS)
select day from t2
where not exists (select 1 from t1 where t1.id = t2.t_id);
```
*Результат*

```
Planning:
  Buffers: shared hit=112 read=12 dirtied=1
Planning Time: 0.707 ms
Execution Time: 5131.234 ms
```

### Задание 4. ускорить запрос "semi-join", добиться времени выполнения < 10sec (запрос со статистикой выполнения)
```
select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```

**Решение**

```
EXPLAIN (ANALYZE, BUFFERS)
select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```
*Результат*

```
Planning:
  Buffers: shared hit=15 read=1 dirtied=1
Planning Time: 0.559 ms
Execution Time: 4948.962 ms
```

Преобразуем запрос

```
EXPLAIN (ANALYZE, BUFFERS)
SELECT t2.day FROM t2 JOIN t1 ON t1.id = t2.t_id
WHERE t2.day > (CURRENT_DATE - '1 months'::interval);
```
*Результат*

```
Planning:
  Buffers: shared hit=21 read=7 dirtied=1
Planning Time: 0.483 ms
Execution Time: 1549.804 ms
```
