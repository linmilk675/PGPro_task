 ## Введение

Сутью тестового задания является установка PostgreSQL 16, запуск нескольких запросов и самое главное - Вы должны сделать оптимизацию нескольких запросов. Точнее Вам нужно объяснить почему запрос выполняется медленно, почему так происходит и как можно это изменить. 

Мы не требуем выполнения всех пунктов задания, будем рады если Вы выполните хотя бы один из пунктов.

Как приступить к выполнению задания:
- Установить на свой компьютер PostgreSQL 16 (желательно одну из последних версий - 16.6 / 16.7)
- Запустите скрипты из параграфа "Скрипт для генерации базы"
- После этого приступайте к задачам 

 ## Задачи

**Важно:** предоставить доказательства для задач 1-4 в виде плана запроса со _статистикой выполнения_.   

### [1] ускорить простой запроc, добиться времени выполнения < 10ms
``` sql
select name from t1 where id = 50000;
```
### [2] ускорить запрос "max + left join", добиться времени выполнения < 10ms
``` sql
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```
### [3] ускорить запрос "anti-join", добиться времени выполнения < 10sec
``` sql
select day from t2 where t_id not in ( select t1.id from t1 );
```
### [4] ускорить запрос "semi-join", добиться времени выполнения < 10sec
``` sql
select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```
### [5] ускорить работу "savepoint + update", добиться постоянной во времени производительности (число транзакций в секунду)

Для начала создайте функцию с помощью следующей команды:
``` bash
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
``` bash
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
``` bash
# check in single session using psql
psql < generate_100_subtrans.sql > generate_100_subtrans.lst
```

И запустите сам тест. 
Он состоит из двух частей: сессии которая ничего не делает и 10 сессий которые генерят нагрузку.
Запустить можно его так:
``` bash
# execute long running transaction
psql -c 'select txid_current(); select pg_sleep(3600);' &
# check using pgbench
pgbench -p 5433 -rn -P1 -c10 -T3600 -M prepared -f generate_100_subtrans.sql 2>&1 > generate_100_subtrans_pgbench.log
```


 ## Скрипт для генерации базы
``` sql
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
