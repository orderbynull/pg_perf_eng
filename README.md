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
 
 Ситуацию можно улучшить, если добавить индекс по полю id: `create unique index on t1(id);`, что позволит прочитать лишь несколько страниц памяти(от корня индексного дерева до его листа) вместо всех страниц памяти.
 
После добавление индекса:
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
 Здесь видно, что было считано всего **4** страницы памяти: **3** страницы индекса(высота btree дерева в данном случае **3**, получено от **pgstattuple**) + **1** страница таблицы(фактическое значение для результата запроса). Все это привело к времени выполнения менее **1ms**.