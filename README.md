> [!WARNING]
> В задании сказано, что требуется предоставить план запроса со статистикой выполнения. Здесь не очень ясно, что именно подразумевается под статистикой - EXPLAIN ANALYZE или множественные запуски с фиксированием результатов выполнения? Какая статистика должна быть для последней задачи с pgbench? Решил что поступлю так: результаты оптимизации буду выводить с EXPLAIN (ANALYZE, TIMING, BUFFERS), а для последней задачи приведу логи результатов выполнения pgbench на несколько сотен строк.
## Задача 1
Первоначальный план запроса:
```sql
postgres=# explain (analyze, timing, buffers) select name from t1 where id = 50000;
                                                 QUERY PLAN                                                 
------------------------------------------------------------------------------------------------------------
 Seq Scan on t1  (cost=0.00..208331.66 rows=1 width=30) (actual time=6.786..694.628 rows=1 loops=1)
   Filter: (id = 50000)
   Rows Removed by Filter: 9999999
   Buffers: shared hit=2054 read=81280
 Planning:
   Buffers: shared hit=45
 Planning Time: 0.099 ms
 JIT:
   Functions: 4
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.299 ms, Inlining 0.000 ms, Optimization 0.111 ms, Emission 1.637 ms, Total 2.047 ms
 Execution Time: 694.958 ms
 ```
 Здесь видно, что было произведено последовательное сканирование всей таблицы, при этом из общего кол-ва страниц(**83334**) занимаемых таблицей, с диска было прочитано **81280** страниц, а из кеша - **2054** страниц. Каждое обращение к диску это драгоценное время, в сумме это дало **694.958 ms**.
 
 Ситуацию можно улучшить, если добавить следующий индекс: `create unique index on t1(id);`. Это позволит прочитать лишь несколько страниц памяти(от корня индексного дерева до его листа + 1) вместо всех страниц памяти:
 ```sql
postgres=# explain (analyze, timing, buffers) select name from t1 where id = 50000;
                                                  QUERY PLAN                                                   
---------------------------------------------------------------------------------------------------------------
 Index Scan using t1_id_idx on t1  (cost=0.43..8.45 rows=1 width=30) (actual time=0.633..0.634 rows=1 loops=1)
   Index Cond: (id = 50000)
   Buffers: shared read=4
 Planning:
   Buffers: shared hit=15 read=1
 Planning Time: 0.132 ms
 Execution Time: 0.642 ms
(7 rows)
 ```
 Здесь видно, что было считано всего **4** страницы памяти: **3** страницы индекса(высота btree в данном случае **3**, получено от **pgstattuple**) + **1** страница таблицы(фактическое значение для результата запроса). Все это привело к времени выполнения менее **1ms**.

## Задача 2
При внимательном взгляде на запрос можно увидеть что `LEFT JOIN` здесь совершенно лишний, так как в выбираемых значениях нет ничего из присоединяемой таблицы. PostgreSQL [умеет](https://wiki.postgresql.org/wiki/What%27s_new_in_PostgreSQL_9.0#Join_Removal) избавляться от джоинов при определенных условиях.

Первоначальный запрос:
```sql
postgres=# explain (analyze, timing, buffers) select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
                                                          QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=385604.98..385604.99 rows=1 width=32) (actual time=2718.499..2718.502 rows=1 loops=1)
   Buffers: shared read=115182, temp read=17854 written=17854
   ->  Hash Left Join  (cost=218278.81..373105.00 rows=4999992 width=9) (actual time=917.417..2505.058 rows=5000000 loops=1)
         Hash Cond: (t2.t_id = t1.id)
         Buffers: shared read=115182, temp read=17854 written=17854
         ->  Seq Scan on t2  (cost=0.00..81847.92 rows=4999992 width=13) (actual time=17.621..461.413 rows=5000000 loops=1)
               Buffers: shared read=31848
         ->  Hash  (cost=208335.00..208335.00 rows=606065 width=4) (actual time=899.218..899.219 rows=625530 loops=1)
               Buckets: 262144  Batches: 4  Memory Usage: 7528kB
               Buffers: shared read=83334, temp written=1374
               ->  Seq Scan on t1  (cost=0.00..208335.00 rows=606065 width=4) (actual time=0.130..817.932 rows=625530 loops=1)
                     Filter: (name ~~ 'a%'::text)
                     Rows Removed by Filter: 9374470
                     Buffers: shared read=83334
 Planning:
   Buffers: shared hit=139 read=32
 Planning Time: 3.440 ms
 JIT:
   Functions: 13
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.540 ms, Inlining 0.000 ms, Optimization 0.728 ms, Emission 16.634 ms, Total 17.903 ms
 Execution Time: 2787.034 ms
(22 rows)
```

Добавим самые простые индексы:
```sql
create unique index on t1(id);
create unique index on t2(t_id);
```
Результат:
```sql
postgres=# explain (analyze, timing, buffers) select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
                                                     QUERY PLAN
--------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=94348.00..94348.01 rows=1 width=32) (actual time=580.301..580.302 rows=1 loops=1)
   Buffers: shared hit=96 read=31752
   ->  Seq Scan on t2  (cost=0.00..81848.00 rows=5000000 width=9) (actual time=0.255..302.427 rows=5000000 loops=1)
         Buffers: shared hit=96 read=31752
 Planning:
   Buffers: shared hit=30 read=2
 Planning Time: 1.072 ms
 Execution Time: 580.386 ms
(8 rows)
```

В надежде задействовать индекс для подсчета `MAX(day)` испробованы разные варианты индексов, например:
```sql
create index on t2(day);
create index on t2(t_id, day);
...
```
Ни один из них не привел к успеху. Левый джоин, судя по всему, заставляет планировать полный просмотр всех строк для рассчета `MAX(day)`.
Пробуем заставить использовать индекс + добавим `shared_buffers` чтобы избавиться от `read=31144` в плане запроса:
```sql
set enable_seqscan to off;

postgres=# explain (analyze, timing, buffers) select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
                                                                    QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=164412.43..164412.44 rows=1 width=32) (actual time=502.343..502.344 rows=1 loops=1)
   Buffers: shared hit=19161
   ->  Index Only Scan using t2_t_id_day_idx2 on t2  (cost=0.43..151912.43 rows=5000000 width=9) (actual time=0.012..299.097 rows=5000000 loops=1)
         Heap Fetches: 0
         Buffers: shared hit=19161
 Planning Time: 0.053 ms
 JIT:
   Functions: 2
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.093 ms, Inlining 0.000 ms, Optimization 0.059 ms, Emission 1.093 ms, Total 1.245 ms
 Execution Time: 502.477 ms
(11 rows)
```

`Index only scan` здесь не помог, потому что `rows=5000000` как было, так и осталось. **Без изменения запроса решить задачу не удалось.** А так, конечно, верный запрос был бы следующий(с индексом на `t2.day DESC`):
```sql
select max(t2.day) from t2;
```

 ## Задача 3
 После запуска запроса(завершения которого так и не удалось дождаться) можно увидеть что для вычисления результата задействуются временные файлы:
```bash
postgres@pg-task-3:~/16/main/base/pgsql_tmp$ ls -laht
total 134M
-rw------- 1 postgres postgres 134M Apr 17 13:07 pgsql_tmp5053.0
```
Статистику по кол-ву записанных байтов во временные файлы сейчас смотреть нет смысла, так как она обновляется после выполнения запроса, а дождаться его выполнения нет разумной возможности.

Наличие временного файла признак того, что значение `work_mem` задано слишком низким:
```sql
postgres=# show work_mem;
 work_mem
----------
 4MB
(1 row)
```

Увеличим и повторим запрос:
```sql
set work_mem to '256MB';

postgres=# explain (analyze, timing, buffers) select day from t2 where t_id not in ( select t1.id from t1 );
                                                       QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------
 Seq Scan on t2  (cost=208404.95..302779.88 rows=2500117 width=9) (actual time=4927.223..4927.224 rows=0 loops=1)
   Filter: (NOT (hashed SubPlan 1))
   Rows Removed by Filter: 5000000
   Buffers: shared hit=4257 read=111007
   SubPlan 1
     ->  Seq Scan on t1  (cost=0.00..183402.36 rows=10001036 width=4) (actual time=0.155..686.237 rows=10000000 loops=1)
           Buffers: shared hit=2144 read=81248
 Planning Time: 0.035 ms
 JIT:
   Functions: 17
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.590 ms, Inlining 0.000 ms, Optimization 0.331 ms, Emission 5.488 ms, Total 6.409 ms
 Execution Time: 4950.830 ms
(13 rows)
```

Запрос выполнен менее чем за 10 секунд, как сказано в задаче. Дальнейшие оптимизации не производил(как минимум можно увеличить `shared_buffers` чтобы избавиться от `read=81248`).

 ## Задача 4
 Запустив запрос без оптимизаций не удалось дождаться его выполнения. Стоит сразу добавить индексы на все поля, которые используются в условиях выборки:
 ```sql
 -- с соблюдением порядка полей
 create unique index on t2(t_id, day);

 create unique index on t1(id);
 ```
 и получаем такой результат:
 ```sql
 postgres=# explain (analyze, timing, buffers) select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
                                                                 QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using t2_t_id_day_idx on t2  (cost=0.44..3963737.08 rows=431769 width=9) (actual time=29.453..1324.387 rows=833613 loops=1)
   Index Cond: (day > to_char(date_trunc('day'::text, (now() - '1 mon'::interval)), 'yyyymmdd'::text))
   Filter: (SubPlan 1)
   Heap Fetches: 157
   Buffers: shared hit=2473433 read=46583 written=4
   SubPlan 1
     ->  Index Only Scan using t1_id_idx on t1  (cost=0.43..8.45 rows=1 width=4) (actual time=0.001..0.001 rows=1 loops=833613)
           Index Cond: (id = t2.t_id)
           Heap Fetches: 12
           Buffers: shared hit=2473431 read=27423 written=2
 Planning:
   Buffers: shared hit=35 read=3 dirtied=1
 Planning Time: 0.221 ms
 JIT:
   Functions: 8
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 0.318 ms, Inlining 8.387 ms, Optimization 10.926 ms, Emission 10.057 ms, Total 29.686 ms
 Execution Time: 1346.267 ms
(18 rows)
 ```
 Согласно условию задача уже решена, так как запрос выполнился менее чем за 10 сек. Но можно продолжить оптимизации: избавиться от `Heap Fetches: 12` и `Buffers: read=27423` создав покрывающий индекс(не забыв выполинть `VACUUM` для `Index only scan`) и увеличив `shared_buffers`:
 ```sql
postgres=# explain (analyze, timing, buffers) select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
                                                                 QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using t2_t_id_day_idx on t2  (cost=0.44..2236557.08 rows=431769 width=9) (actual time=31.837..1259.483 rows=833613 loops=1)
   Index Cond: (day > to_char(date_trunc('day'::text, (now() - '1 mon'::interval)), 'yyyymmdd'::text))
   Filter: (SubPlan 1)
   Heap Fetches: 0
   Buffers: shared hit=2520003
   SubPlan 1
     ->  Index Only Scan using t1_id_id1_idx on t1  (cost=0.43..4.45 rows=1 width=4) (actual time=0.001..0.001 rows=1 loops=833613)
           Index Cond: (id = t2.t_id)
           Heap Fetches: 0
           Buffers: shared hit=2500842
 Planning Time: 0.129 ms
 JIT:
   Functions: 9
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 0.357 ms, Inlining 9.064 ms, Optimization 11.861 ms, Emission 10.835 ms, Total 32.116 ms
 Execution Time: 1280.612 ms
(16 rows)
 ```

 ## Задача 5 (не верно, переделать)
 > [!WARNING]
> В задании сказано добиться постоянной во времени производительности, но не сказано что значит "постоянной". Так же не ясно можно ли жертвовать производительностью в угоду стабильности. Так что буду исходить из чисто "спортивного" интереса - добиться поставленной задачи не смотря ни на что.

Сразу делаем запуск тестовых команд без оптимизаций, получаем такую картину:
```bash
# psql -c 'select txid_current(); select pg_sleep(3600);' &
# pgbench -p 5432 -rn -P1 -c10 -T3600 -M prepared -f generate_100_subtrans.sql

pgbench (16.7 (Ubuntu 16.7-1.pgdg22.04+1))
progress: 1.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 2.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 3.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 4.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 5.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 6.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 7.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 8.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 9.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 10.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 11.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 12.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 13.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
```

Все выполняется очень медленно, потому что:
```sql
postgres=# explain (analyze, timing, buffers) update t1 set name = name where id = 10;
                                                 QUERY PLAN
-------------------------------------------------------------------------------------------------------------
 Update on t1  (cost=0.00..233702.30 rows=0 width=0) (actual time=1050.363..1050.363 rows=0 loops=1)
   Buffers: shared hit=16133 read=93172 dirtied=35 written=2
   ->  Seq Scan on t1  (cost=0.00..233702.30 rows=1 width=36) (actual time=475.060..1049.656 rows=1 loops=1)
         Filter: (id = 10)
         Rows Removed by Filter: 9999999
         Buffers: shared hit=16130 read=93172 dirtied=34 written=2
 Planning:
   Buffers: shared hit=11
 Planning Time: 3.841 ms
 JIT:
   Functions: 5
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.563 ms, Inlining 0.000 ms, Optimization 0.167 ms, Emission 5.437 ms, Total 6.167 ms
 Execution Time: 1131.886 ms
 ```

 Здесь требуется индекс: `create unique index on t1(id);`, что приводит в такому результату:
```sql
postgres=# explain (analyze, timing, buffers) update t1 set name = name where id = 10;
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 Update on t1  (cost=0.43..8.45 rows=0 width=0) (actual time=0.025..0.026 rows=0 loops=1)
   Buffers: shared hit=6
   ->  Index Scan using t1_id_idx on t1  (cost=0.43..8.45 rows=1 width=36) (actual time=0.014..0.015 rows=1 loops=1)
         Index Cond: (id = 10)
         Buffers: shared hit=4
 Planning Time: 0.036 ms
 Execution Time: 0.036 ms
```

Теперь пробуем нагрузочный тест:
```bash
# psql -c 'select txid_current(); select pg_sleep(3600);' &
# pgbench -p 5432 -rn -P1 -c10 -T3600 -M prepared -f generate_100_subtrans.sql

pgbench (16.7 (Ubuntu 16.7-1.pgdg22.04+1))
progress: 1.0 s, 88.0 tps, lat 103.974 ms stddev 22.038, 0 failed
progress: 2.0 s, 116.0 tps, lat 90.536 ms stddev 6.846, 0 failed
progress: 3.0 s, 101.0 tps, lat 94.553 ms stddev 6.125, 0 failed
progress: 4.0 s, 109.0 tps, lat 95.977 ms stddev 7.123, 0 failed
progress: 5.0 s, 138.0 tps, lat 72.544 ms stddev 23.334, 0 failed
progress: 6.0 s, 225.0 tps, lat 44.368 ms stddev 4.865, 0 failed
progress: 7.0 s, 220.0 tps, lat 45.545 ms stddev 5.225, 0 failed
progress: 8.0 s, 216.0 tps, lat 45.762 ms stddev 5.010, 0 failed
progress: 9.0 s, 220.0 tps, lat 45.417 ms stddev 4.751, 0 failed
progress: 10.0 s, 224.0 tps, lat 44.673 ms stddev 4.871, 0 failed
progress: 11.0 s, 224.0 tps, lat 44.593 ms stddev 4.498, 0 failed
progress: 12.0 s, 216.0 tps, lat 46.286 ms stddev 6.072, 0 failed
progress: 13.0 s, 223.0 tps, lat 45.036 ms stddev 4.912, 0 failed
progress: 14.0 s, 223.0 tps, lat 45.242 ms stddev 5.255, 0 failed
progress: 15.0 s, 221.0 tps, lat 44.724 ms stddev 5.032, 0 failed
progress: 16.0 s, 220.0 tps, lat 45.352 ms stddev 4.695, 0 failed
progress: 17.0 s, 222.0 tps, lat 44.810 ms stddev 4.932, 0 failed
progress: 18.0 s, 228.0 tps, lat 44.395 ms stddev 4.887, 0 failed
progress: 19.0 s, 226.0 tps, lat 44.124 ms stddev 4.778, 0 failed
progress: 20.0 s, 226.0 tps, lat 44.226 ms stddev 4.993, 0 failed
progress: 21.0 s, 227.0 tps, lat 44.080 ms stddev 5.181, 0 failed
progress: 22.0 s, 219.0 tps, lat 45.393 ms stddev 4.960, 0 failed
progress: 23.0 s, 224.0 tps, lat 44.742 ms stddev 5.345, 0 failed
progress: 24.0 s, 226.0 tps, lat 44.234 ms stddev 5.118, 0 failed
progress: 25.0 s, 231.0 tps, lat 43.721 ms stddev 4.593, 0 failed
progress: 26.0 s, 222.0 tps, lat 44.591 ms stddev 4.963, 0 failed
progress: 27.0 s, 222.0 tps, lat 44.993 ms stddev 5.263, 0 failed
progress: 28.0 s, 224.0 tps, lat 44.662 ms stddev 4.582, 0 failed
progress: 29.0 s, 230.0 tps, lat 43.740 ms stddev 4.769, 0 failed
progress: 30.0 s, 223.0 tps, lat 44.545 ms stddev 4.877, 0 failed
progress: 31.0 s, 224.0 tps, lat 44.505 ms stddev 5.260, 0 failed
progress: 32.0 s, 218.0 tps, lat 45.940 ms stddev 4.898, 0 failed
```

Кажется, tps действительно не стабилен. Попробуем посмотреть на взятые блокировки:
```sql
postgres=# select count(*) from pg_locks;
 count
-------
   611
(1 row)
```

Могу предположить, что нестабильный tps происходит из-за конкуренции между транзакциями в разных сессиях. Есть надежный(в данном "спортивном" случае) способ это исправить - поставить уровень изоляции `SERIALIZABLE`. 

Запускаем нагрузочный тест заново:
```bash
# psql -c 'select txid_current(); select pg_sleep(3600);' &
# pgbench -p 5432 -rn -P1 -c10 -T3600 -M prepared -f generate_100_subtrans.sql


```

wal_buffers
max_wal_size
