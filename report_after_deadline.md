*Замечание* Использована версия PostgreSQL 16.8

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
### Задание 5. ускорить работу "savepoint + update", добиться постоянной во времени производительности (число транзакций в секунду)

Для начала создайте функцию с помощью следующей команды:

```
# generate function random with two arguments
psql -X -q <<'EOF'
create or replace function random(left bigint, right bigint) returns bigint
as $$
 select trunc(random.left + random()*(random.right - random.left))::bigint;
$$                                                
language sql;
EOF
```

Сгенерите SQL файл, который будет создавать нагрузку на базу:

```
# generate 100 subtransactions
psql -X -q > generate_100_subtrans.sql <<EOF
select '\\set id random(1,10000000)'
union all
select 'BEGIN;'
union all
select 'savepoint v' || v.id || ';'                || E'\n' 
    || 'update t1 set name = name where id = :id;' || E'\n'
from generate_series(1,100) v(id)
union all
select E'COMMIT;\n' \g (tuples_only=true format=unaligned)
EOF
```

Проверьте работоспособность скрипта:
```
# check in single session using psql
psql < generate_100_subtrans.sql > generate_100_subtrans.lst
```

И запустите сам тест. Он состоит из двух частей: сессии которая ничего не делает и 10 сессий которые генерят нагрузку. Запустить можно его так:

```
# execute long running transaction
psql -c 'select txid_current(); select pg_sleep(3600);' &
# check using pgbench
pgbench -p 5433 -rn -P1 -c10 -T3600 -M prepared -f generate_100_subtrans.sql 2>&1 > generate_100_subtrans_pgbench.log
```

**Решение**

Так как я, к сожалению, делала всё на винде, моя жизнь стала веселее 

Создаём sql файл create_random_function.sql

```
create or replace function random(left bigint, right bigint) returns bigint
as $$
 select trunc(random.left + random()*(random.right - random.left))::bigint;
$$
language sql;
```

```
C:\Program Files\PostgreSQL\16\bin>psql -U postgres -d postgres -f C:\Users\Malbolge\AppData\Roaming\DBeaverData\workspace6\General\Scripts\create_random_function.sql
Пароль пользователя postgres:
CREATE FUNCTION
```
Создаём sql файл generate_script.sql

```
select '\\set id random(1,10000000)'
union all
select 'BEGIN;'
union all
select 'savepoint v' || v.id || ';'                || E'\n' 
    || 'update t1 set name = name where id = :id;' || E'\n'
from generate_series(1,100) v(id)
union all
select E'COMMIT;\n' \g (tuples_only=true format=unaligned)
```

```
C:\Program Files\PostgreSQL\16\bin>psql -U postgres -d postgres -q -X -f C:\Users\Malbolge\AppData\Roaming\DBeaverData\workspace6\General\Scripts\generate_script.sql > C:\Users\Malbolge\AppData\Roaming\DBeaverData\workspace6\General\Scripts\generate_100_subtrans.sql
```

Проверка работоспособности скрипта

```
C:\Program Files\PostgreSQL\16\bin>psql -U postgres -d postgres < C:\Users\Malbolge\AppData\Roaming\DBeaverData\workspace6\General\Scripts\generate_100_subtrans.sql > C:\Users\Malbolge\AppData\Roaming\DBeaverData\workspace6\General\Scripts\generate_100_subtrans.lst
```

*Результат*
```
неверная команда \
Р?РЁР?Р'Р?Р?:  Р?С?РёР+РєР° С?РёР?С'Р°РєС?РёС?Р° (РїС?РёР?РчС?Р?Р?Рч РїР?Р>Р?Р¶РчР?РёРч: ":")
LINE 1: update t1 set name = name where id = :id;
```
(и на 100 умножить)

```
select 'BEGIN;'
union all
select 'savepoint v' || v.id || ';'                || E'\n' 
    || 'update t1 set name = name where id = random(1,10000000);' || E'\n'
from generate_series(1,100) v(id)
union all
select E'COMMIT;\n' \g (tuples_only=true format=unaligned)
```

Запускаем долгую транзакцию

```
C:\Program Files\PostgreSQL\16\bin>psql -U postgres -d postgres -c "select txid_current(); select pg_sleep(3600);"
```

Запускаем нагрузочный тест
```
C:\Program Files\PostgreSQL\16\bin>pgbench -p 5432 -rn -P1 -c10 -T3600 -M prepared -f C:\Users\Malbolge\AppData\Roaming\DBeaverData\workspace6\General\Scripts\generate_100_subtrans.sql -U postgres -d postgres > C:\Users\Malbolge\AppData\Roaming\DBeaverData\workspace6\General\Scripts\generate_100_subtrans_pgbench.log 2>&1
```

