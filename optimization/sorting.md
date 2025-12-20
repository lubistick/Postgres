# Сортировка

Индексный доступ автоматически возвращает строки, отсортированные по проиндексированному столбцу:
```sql
EXPLAIN (costs off) SELECT * FROM flights ORDER BY flight_id;

                QUERY PLAN                
------------------------------------------
 Index Scan using flights_pkey on flights
(1 row)
```

Но если попросить сервер отсортировать данные по столбцу без индекса, потребуется два отдельных шага: получение данных и сортировка:
```sql
EXPLAIN (costs off) SELECT * FROM flights ORDER BY status;

        QUERY PLAN         
---------------------------
 Sort
   Sort Key: status
   ->  Seq Scan on flights
(3 rows)
```

Дальше мы будем разбираться, как устроена сортировка в узле `Sort`.

В распоряжении планировщика имеется несколько методов сортировки:
- сортировка в памяти
- внешняя сортировка
- инкрементальная сортировка


## Сортировка в памяти

В идеальном случае набор строк, подлежащий сортировке, целиком помещается в память, ограниченную параметром `work_mem`.
Тогда планировщик используется один из двух алгоритмов:
- быстрая сортировка `quicksort`
- частичная пирамидальная сортировка `top-N heapsort`


### Быстрая сортировка

Посмотрим на быструю сортировку:

```sql
EXPLAIN (analyze, timing off, summary off)
SELECT * FROM seats ORDER BY seat_no;

                                          QUERY PLAN                                          
----------------------------------------------------------------------------------------------
 Sort  (cost=90.93..94.28 rows=1339 width=15) (actual rows=1339.00 loops=1)
   Sort Key: seat_no
   Sort Method: quicksort  Memory: 90kB
   Buffers: shared read=8
   ->  Seq Scan on seats  (cost=0.00..21.39 rows=1339 width=15) (actual rows=1339.00 loops=1)
         Buffers: shared read=8
 Planning:
   Buffers: shared hit=43 read=2
(8 rows)
```

Узел `Sort Method` имеет значение `quicksort` - был применен алгоритм быстрой сортировки.
В той же строке указан объем использованной оперативной памяти `Memory: 90kB`.

Чтобы начать выдавать данные, узлу `Sort` нужно полностью отсортировать набор строк.
Поэтому начальная стоимость узла включает в себя полную стоимость чтения таблицы.
В общем случае сложность быстрой сортировки равна O(M logM), где M - число строк в исходном наборе.


### Частичная пирамидальная сортировка

Если набор строк ограничен, планировщик может переключиться на частичную сортировку.
Обратим внимание, что стоимость запроса снизилась и для сортировки потребовалось меньше памяти:

```sql
EXPLAIN (analyze, timing off, summary off)
SELECT * FROM seats ORDER BY seat_no LIMIT 100;

                                             QUERY PLAN                                             
----------------------------------------------------------------------------------------------------
 Limit  (cost=72.57..72.82 rows=100 width=15) (actual rows=100.00 loops=1)
   Buffers: shared hit=8
   ->  Sort  (cost=72.57..75.91 rows=1339 width=15) (actual rows=100.00 loops=1)
         Sort Key: seat_no
         Sort Method: top-N heapsort  Memory: 32kB
         Buffers: shared hit=8
         ->  Seq Scan on seats  (cost=0.00..21.39 rows=1339 width=15) (actual rows=1339.00 loops=1)
               Buffers: shared hit=8
(8 rows)
```

Узел `Sort Method` имеет значение `top-N heapsort`.
Грубо говоря, вместо полной сортировки здесь 100 раз находится минимальное значение.
В общем случае сложность алгоритма `top-N heapsort` ниже, чем для быстрой сортировки.
Она равна O(M logN), где M - число строк в исходном наборе, а N - количество первых строк, которые надо получить.


## Внешняя сортировка

Если набор строк не помещается в оперативную память целиком, используется алгоритм внешней сортировки.
Набор строк читается в память, пока есть возможность, затем сортируется и записывается во временный файл на диске.
Процедура повторяется столько раз, сколько необходимо, чтобы записать все данные в файлы, каждый из которых по отдельности отсортирован.
Далее файлы соединяются с сохранением порядка строк - сравнивается первая строка каждого файла, выбирается минимальная (максимальная), возвращается как часть результата, берется следующая строка и т.д. Результат соединения файлов может быть записан в новый временный файл и так продолжается, пока не соединяться все временные файлы.

Пример плана с внешней сортировкой:

```sql
EXPLAIN (analyze, buffers, timing off, summary off)
SELECT * FROM flights ORDER BY scheduled_departure;

                                              QUERY PLAN                                              
------------------------------------------------------------------------------------------------------
 Sort  (cost=31883.96..32421.12 rows=214867 width=63) (actual rows=214867.00 loops=1)
   Sort Key: scheduled_departure
   Sort Method: external merge  Disk: 17120kB
   Buffers: shared hit=2627, temp read=2140 written=2145
   ->  Seq Scan on flights  (cost=0.00..4772.67 rows=214867 width=63) (actual rows=214867.00 loops=1)
         Buffers: shared hit=2624
 Planning:
   Buffers: shared hit=12 read=1
(8 rows)
```

Узел `Sort Method` имеет значение `external merge` - внешняя сортировка.
Также выводится, сколько места потребовалось на диске для временных файлов: `Disk: 17120kB`.
Метрики `Buffers` показывают количество прочитанных блоков временных файлов `temp read` и количество записанных блоков `temp... written`.

Увеличим значение `work_mem`:

```sql
SET work_mem = '48MB';

SET
```

Повторим запрос:

```sql
EXPLAIN (analyze, buffers, timing off, summary off)
SELECT * FROM flights ORDER BY scheduled_departure;

                                              QUERY PLAN                                              
------------------------------------------------------------------------------------------------------
 Sort  (cost=23802.46..24339.62 rows=214867 width=63) (actual rows=214867.00 loops=1)
   Sort Key: scheduled_departure
   Sort Method: quicksort  Memory: 24482kB
   Buffers: shared hit=2624
   ->  Seq Scan on flights  (cost=0.00..4772.67 rows=214867 width=63) (actual rows=214867.00 loops=1)
         Buffers: shared hit=2624
(6 rows)
```

Теперь все строки поместились в память, и планировщик выбрал более дешевую быструю сортировку.

Вернем значение по умолчанию:

```sql
RESET work_mem;

RESET
```


## Инкрементальная сортировка

Если набор данных требуется отсортировать по ключам K1... Km... Kn, и при этом известно,
что набор уже отсортирован по первым "m" ключам, можно не пересортировывать все данные заново.
Достаточно разбить набор данных на группы, имеющие одинаковые значения начальных ключей K1... Km,
и затем отсортировать отдельно каждую из групп по оставшимся ключам.
Такой способ называется инкрементальной сортировкой.

Алгоритм обрабатывает относитально крупные группы строк отдельно, а небольшие объединяет и сортирует полностью.
Инкрементальная сортировка может использовать как сортировку в памяти, так и внешнюю сортировку, если не хватает объема `work_mem`.

Создадим индекс на таблице "bookings":

```sql
CREATE INDEX ON bookings(total_amount);

CREATE INDEX
```

Посмотрим на пример инкрементальной сортировки:

```sql
EXPLAIN (analyze, buffers off, costs off, timing off, summary off)
SELECT * FROM bookings ORDER BY total_amount, book_date;

                                           QUERY PLAN                                           
------------------------------------------------------------------------------------------------
 Incremental Sort (actual rows=2111110.00 loops=1)
   Sort Key: total_amount, book_date
   Presorted Key: total_amount
   Full-sort Groups: 2823  Sort Method: quicksort  Average Memory: 27kB  Peak Memory: 27kB
   Pre-sorted Groups: 2625  Sort Method: quicksort  Average Memory: 1952kB  Peak Memory: 2014kB
   ->  Index Scan using bookings_total_amount_idx on bookings (actual rows=2111110.00 loops=1)
         Index Searches: 1
(7 rows)
```

Здесь данные, полученные из таблицы "bookings" по только что созданному индексу "bookings_total_amount_idx",
уже отсортированы по столбцу "total_amount" (`Presorted Key`), поэтому остается доупорядочить строки по столбцу "book_date".

Строка `Pre-sorted Groups` относится к крупным группам, которые досортировывались по столбцу "book_date".
Строка `Full-sort Groups` - к небольшим группам, которые были объединены и отсортированы полностью.
В примере все группы поместились в выделенную память и применялась быстрая сортировка.

Уменьшим `work_mem`:

```sql
SET work_mem = '128 kB';

SET
```

Повторим запрос:

```sql
EXPLAIN (analyze, buffers off, costs off, timing off, summary off)
SELECT * FROM bookings ORDER BY total_amount, book_date;

                                                                     QUERY PLAN                                                                      
-----------------------------------------------------------------------------------------------------------------------------------------------------
 Incremental Sort (actual rows=2111110.00 loops=1)
   Sort Key: total_amount, book_date
   Presorted Key: total_amount
   Full-sort Groups: 2823  Sort Method: quicksort  Average Memory: 27kB  Peak Memory: 27kB
   Pre-sorted Groups: 2625  Sort Methods: quicksort, external merge  Average Memory: 0kB  Peak Memory: 41kB  Average Disk: 1017kB  Peak Disk: 1056kB
   ->  Index Scan using bookings_total_amount_idx on bookings (actual rows=2111110.00 loops=1)
         Index Searches: 1
(7 rows)
```

Теперь при нехватке памяти, для некоторых крупных групп пришлось применить внешнюю сортировку с использованием временных файлов
`Pre-sorted Groups... external merge`.

```sql
RESET work_mem;

RESET
```


## Оконные функции

Посмотрим план запроса, вычисляющего сумму бронирований нарастающим итогом:

```sql
EXPLAIN SELECT *, sum(total_amount) OVER (ORDER BY book_date) FROM bookings;

                                   QUERY PLAN                                    
---------------------------------------------------------------------------------
 WindowAgg  (cost=342956.76..379901.10 rows=2111110 width=53)
   Window: w1 AS (ORDER BY book_date)
   ->  Sort  (cost=342956.67..348234.45 rows=2111110 width=21)
         Sort Key: book_date
         ->  Seq Scan on bookings  (cost=0.00..34599.10 rows=2111110 width=21)
 JIT:
   Functions: 7
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(8 rows)
```

Оконная функция вычисляется в узле `WindowAgg`, который получает отсортированные данные от дочернего узла `Sort`.

Добавление к запросу других оконных функций, использующих тот же порядок строк (не обязательно с совпадающим окном), а также предложение `ORDER BY` не приводит к появлению лишних сортировок:

```sql
EXPLAIN SELECT *,
  sum(total_amount) OVER (ORDER BY book_date),
  avg(total_amount) OVER (ORDER BY book_date ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING)
FROM bookings ORDER BY book_date;

                                            QUERY PLAN                                             
---------------------------------------------------------------------------------------------------
 WindowAgg  (cost=342956.89..411567.75 rows=2111110 width=85)
   Window: w2 AS (ORDER BY book_date ROWS BETWEEN '3'::bigint PRECEDING AND '3'::bigint FOLLOWING)
   ->  WindowAgg  (cost=342956.76..379901.10 rows=2111110 width=53)
         Window: w1 AS (ORDER BY book_date)
         ->  Sort  (cost=342956.67..348234.45 rows=2111110 width=21)
               Sort Key: book_date
               ->  Seq Scan on bookings  (cost=0.00..34599.10 rows=2111110 width=21)
 JIT:
   Functions: 14
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(10 rows)
```

Здесь два узла `WindowAgg` (каждый для своего окна), но общий узел `Sort`.
Стоимость запроса увеличилась, но лишь немного.

Конечно, оконные функции, требующие разного порядка строк, вынуждают сервер пересортировывать данные:

```sql
EXPLAIN SELECT *,
  sum(total_amount) OVER (ORDER BY book_date),
  count(*) OVER (ORDER BY book_ref)
FROM bookings;

                                                QUERY PLAN                                                 
-----------------------------------------------------------------------------------------------------------
 WindowAgg  (cost=422784.39..459728.73 rows=2111110 width=61)
   Window: w2 AS (ORDER BY book_date)
   ->  Sort  (cost=422784.30..428062.08 rows=2111110 width=29)
         Sort Key: book_date
         ->  WindowAgg  (cost=0.48..99992.73 rows=2111110 width=29)
               Window: w1 AS (ORDER BY book_ref)
               ->  Index Scan using bookings_pkey on bookings  (cost=0.43..68326.08 rows=2111110 width=21)
 JIT:
   Functions: 8
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(10 rows)
```

В этом примере нижний узел `WindowAgg`, соответствующий функции `count`, получает упорядоченный набор строк по индексу.
А для верхнего узла `WindowAgg`, соответствующего функции `sum`, строки переупорядочивает узел `Sort`.

Сортировка в оконных функциях может использовать любые методы, рассмотренные выше.
Например, быструю сортировку:

```sql
EXPLAIN (analyze, buffers, timing off, summary off)
SELECT *, count(*) OVER (ORDER BY seat_no) FROM seats;

                                             QUERY PLAN                                             
----------------------------------------------------------------------------------------------------
 WindowAgg  (cost=90.98..114.36 rows=1339 width=23) (actual rows=1339.00 loops=1)
   Window: w1 AS (ORDER BY seat_no)
   Storage: Memory  Maximum Storage: 17kB
   Buffers: shared read=8
   ->  Sort  (cost=90.93..94.28 rows=1339 width=15) (actual rows=1339.00 loops=1)
         Sort Key: seat_no
         Sort Method: quicksort  Memory: 90kB
         Buffers: shared read=8
         ->  Seq Scan on seats  (cost=0.00..21.39 rows=1339 width=15) (actual rows=1339.00 loops=1)
               Buffers: shared read=8
 Planning:
   Buffers: shared hit=32 read=1 dirtied=1
(12 rows)
```

Или внешнюю:

```sql
EXPLAIN (analyze, buffers, timing off, summary off)
SELECT *, sum(total_amount) OVER (ORDER BY book_date) FROM bookings;

                                                   QUERY PLAN                                                   
----------------------------------------------------------------------------------------------------------------
 WindowAgg  (cost=342956.76..379901.10 rows=2111110 width=53) (actual rows=2111110.00 loops=1)
   Window: w1 AS (ORDER BY book_date)
   Storage: Memory  Maximum Storage: 17kB
   Buffers: shared hit=282 read=13206, temp read=16488 written=16521
   ->  Sort  (cost=342956.67..348234.45 rows=2111110 width=21) (actual rows=2111110.00 loops=1)
         Sort Key: book_date
         Sort Method: external merge  Disk: 65968kB
         Buffers: shared hit=282 read=13206, temp read=16488 written=16521
         ->  Seq Scan on bookings  (cost=0.00..34599.10 rows=2111110 width=21) (actual rows=2111110.00 loops=1)
               Buffers: shared hit=282 read=13206
 JIT:
   Functions: 7
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(13 rows)
```


