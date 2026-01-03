# Группировка

Планировщик будет делать группировку в случаях:
- явно указать предложение `GROUP BY`
- при устранении дубликатов с помощью `DISTINCT` или с помощью операторов работы с множествами `UNION`, `INTERSECT`, `EXCEPT`
- для оконных функций с предложением `PARTITION BY`

## Группировка хешированием

Посмотрим пример:

```sql
EXPLAIN SELECT aircraft_code, count(*) FROM seats GROUP BY aircraft_code;

                          QUERY PLAN                           
---------------------------------------------------------------
 HashAggregate  (cost=28.09..28.18 rows=9 width=12)
   Group Key: aircraft_code
   ->  Seq Scan on seats  (cost=0.00..21.39 rows=1339 width=4)
(3 rows)
```

Смысл хеширования в следующем - создается хеш таблица, со счетчиком.
Для каждой строки запроса рассчитывается хеш от значения колонок после `GROUP BY`, т.е. от "aircraft_code".
Если хеша нет в таблице, то создается запись с хешом и значением счетчика 0.
Если хеш есть, то просто увеличивается счетчик.
Таким образом в хеш таблице собираются все уникальные значения.
Алгоритм возвращает сами значения и их количество.

`HashAggregate` - узел, который отвечает за агрегацию методом хеширования.
Обратите внимание на начальную стоимость: результаты начинают выдаваться только после построения хеш-таблицы по всем строкам.

Выполнив запрос, можно увидеть объем памяти, выделенной под хеш-таблицу:

```sql
EXPLAIN (analyze, buffers off, timing off, costs off)
SELECT aircraft_code, count(*) FROM seats GROUP BY aircraft_code;

                      QUERY PLAN                       
-------------------------------------------------------
 HashAggregate (actual rows=9.00 loops=1)
   Group Key: aircraft_code
   Batches: 1  Memory Usage: 32kB
   ->  Seq Scan on seats (actual rows=1339.00 loops=1)
 Planning Time: 0.083 ms
 Execution Time: 0.445 ms
(6 rows)
```

`Batches: 1` означает, что вся хеш-таблица поместилась в оперативную память.
`Memory Usage: 32kB` - объем оперативной памяти, который потребовался.


### Группировка хешированием и пакеты

Рассмотрим план другого запроса, с группировкой по большой таблице:

```sql
EXPLAIN SELECT book_ref, count(*) FROM tickets GROUP BY book_ref;

                                   QUERY PLAN                                    
---------------------------------------------------------------------------------
 HashAggregate  (cost=244868.03..282286.58 rows=1437280 width=15)
   Group Key: book_ref
   Planned Partitions: 16
   ->  Seq Scan on tickets  (cost=0.00..78938.57 rows=2949857 width=7)
 JIT:
   Functions: 3
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(7 rows)
```

В плане появилось предложение `Planned Partitions` - оптимизатор предполагает, что хеш-таблица не поместится в выделенную память и придется разбивать ее на пакеты.

Выполним запрос:

```sql
EXPLAIN (analyze, buffers off, timing off, costs off)
SELECT book_ref, count(*) FROM tickets GROUP BY book_ref;

                         QUERY PLAN                         
------------------------------------------------------------
 HashAggregate (actual rows=2111110.00 loops=1)
   Group Key: book_ref
   Batches: 81  Memory Usage: 8601kB  Disk Usage: 80456kB
   ->  Seq Scan on tickets (actual rows=2949857.00 loops=1)
 Planning Time: 0.092 ms
 Execution Time: 2886.284 ms
(6 rows)
```

Общее количество пакетов `Batches` увеличилось по сравнению с расчетным.
Тут же показано, сколько места заняли временные файлы на диске `Disk Usage: 80456kB`.

При группировке хешированием выделяется `work_mem` умножить на `hash_mem_multiplier` оперативной памяти.


## Группировка сортировкой

Исключение дубликатов и группировка могут выполняться не только путем хеширования, но и с помощью сортировки.
В этом случае требуется набор данных, предварительно отсортированный по полям группировки.

Рассмотрим запрос:

```sql
EXPLAIN (costs off)
SELECT ticket_no, count(ticket_no) FROM ticket_flights GROUP BY ticket_no;

                            QUERY PLAN                             
-------------------------------------------------------------------
 GroupAggregate
   Group Key: ticket_no
   ->  Index Only Scan using ticket_flights_pkey on ticket_flights
(3 rows)
```

Узел `Index Only Scan` получает отсортированный набор данных.
В узле `GroupAggregate` происходит группировка отсортированного набора.

Запрос с `DISTINCT` использует для устранения дубликатов узел `Unique`:

```sql
EXPLAIN (costs off)
SELECT DISTINCT ticket_no FROM ticket_flights ORDER BY ticket_no;

                            QUERY PLAN                             
-------------------------------------------------------------------
 Unique
   ->  Index Only Scan using ticket_flights_pkey on ticket_flights
(2 rows)
```

Поскольку узел возвращает результаты, упорядоченные так же, как требует предложение `ORDER BY`, дополнительная сортировка не требуется.
В отличие от узла `HashAggregate`, оба узла `Unique` и `GroupAggregate` возвращают упорядоченные данные.


## Оконные функции

Если определение окна включает конструкцию `PARTITION BY`, оконная функция вычисляется на основе групп строк (аналогично группировке `GROUP BY`).

