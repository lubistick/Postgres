# Сканирование по битовой карте

При индексном доступе табличные страницы могут читаться несколько раз.
Чтобы этого избежать, существует метод сканирования по битовой карте, который выполняется в две операции:
- сначала сканируется индекс (`Bitmap Index Scan`) и в оперативной памяти строится битовая карта со ссылками на табличные страницы и на версии строк, которые удовлетворяют условиям и должны быть прочитаны
- когда битовая карта готова, сканируется таблица (`Bitmap Heap Scan`) - читаются нужные версии строк из страниц


Будем рассматривать таблицу бронирований bookings.

Создадим на ней два дополнительных индекса:

```sql
CREATE INDEX ON bookings(book_date);

CREATE INDEX
```

```sql
CREATE INDEX ON bookings(total_amount);

CREATE INDEX
```

Посмотрим, какой метод доступа будет выбран для поиска диапазона:
```sql
EXPLAIN SELECT * FROM bookings WHERE total_amount < 10000;

                                          QUERY PLAN                                           
-----------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings  (cost=1180.01..15413.42 rows=62913 width=21)
   Recheck Cond: (total_amount < '10000'::numeric)
   ->  Bitmap Index Scan on bookings_total_amount_idx  (cost=0.00..1164.28 rows=62913 width=0)
         Index Cond: (total_amount < '10000'::numeric)
(4 rows)
```

Выбран метод доступа `Bitmap Scan`. Он состоит из двух узлов:
- `Bitmap Index Scan` - читает индекс и строит битовую карту
- `Bitmap Heap Scan` - читает табличные страницы, используя построенную карту

Обратите внимание, что карта должна быть построена полностью, прежде чем ее можно будет использовать.


## Объединение битовых карт

Кроме того, что битовая карта позволяет избежать повторных чтений табличных страниц, с ее помощью можно объединить несколько условий.

```sql
EXPLAIN (costs off)
SELECT * FROM bookings WHERE total_amount < 10000 OR total_amount > 100000;

                                        QUERY PLAN                                         
-------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings
   Recheck Cond: ((total_amount < '10000'::numeric) OR (total_amount > '100000'::numeric))
   ->  BitmapOr
         ->  Bitmap Index Scan on bookings_total_amount_idx
               Index Cond: (total_amount < '10000'::numeric)
         ->  Bitmap Index Scan on bookings_total_amount_idx
               Index Cond: (total_amount > '100000'::numeric)
(7 rows)
```

Здесь сначала были построены две битовые карты - по одной на каждое условие, а затем объединены побитовой операцией "или".

Таким же образом могут быть использованы и разные индексы.
```sql
EXPLAIN (costs off)
SELECT * FROM bookings WHERE total_amount < 10000 OR book_date = bookings.now() - INTERVAL '1 day';

                                                                  QUERY PLAN                                                                   
-----------------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings
   Recheck Cond: ((total_amount < '10000'::numeric) OR (book_date = ('2017-08-15 15:00:00+00'::timestamp with time zone - '1 day'::interval)))
   ->  BitmapOr
         ->  Bitmap Index Scan on bookings_total_amount_idx
               Index Cond: (total_amount < '10000'::numeric)
         ->  Bitmap Index Scan on bookings_book_date_idx
               Index Cond: (book_date = ('2017-08-15 15:00:00+00'::timestamp with time zone - '1 day'::interval))
(7 rows)
```


## Неточные фрагменты

Битовая карта хранится в локальной памяти обслуживающего процесса и под ее хранение выделяется `work_mem` памяти.
Временные файлы никогда не используются.
Если карта перестает помещаться в `work_mem`, часть ее фрагментов огрубляется - каждый бит начинает соответствовать целой странице, а не отдельной версии строки.
Стоимость обработки таких фрагментов увеличивается.

Узел `Bitmap Heap Scan` показывает условие перепроверки `Recheck Cond`.
Сама же перепроверка выполняется не всегда, а только при потере точности, когда битовая карта не помещается в память.

Будем искать бронирования с суммой до 5 тысяч, сделанные за последний месяц:

```sql
SELECT bookings.now() - INTERVAL '1 months';

        ?column?        
------------------------
 2017-07-15 15:00:00+00
(1 row)
```

Создадим связанную переменную `$1` для полученного значения:

```sql
\bind '2017-07-15 15:00:00+00'
```

Выполняем запрос:
```sql
EXPLAIN (analyze, costs off, timing off)
SELECT count(*) FROM bookings WHERE total_amount < 5000 AND book_date > $1;

                                                          QUERY PLAN                                                           
-------------------------------------------------------------------------------------------------------------------------------
 Aggregate (actual rows=1.00 loops=1)
   Buffers: shared hit=9 read=410
   ->  Bitmap Heap Scan on bookings (actual rows=129.00 loops=1)
         Recheck Cond: ((total_amount < '5000'::numeric) AND (book_date > '2017-07-15 15:00:00+00'::timestamp with time zone))
         Heap Blocks: exact=129
         Buffers: shared hit=9 read=410
         ->  BitmapAnd (actual rows=0.00 loops=1)
               Buffers: shared hit=3 read=287
               ->  Bitmap Index Scan on bookings_total_amount_idx (actual rows=1471.00 loops=1)
                     Index Cond: (total_amount < '5000'::numeric)
                     Index Searches: 1
                     Buffers: shared hit=3 read=4
               ->  Bitmap Index Scan on bookings_book_date_idx (actual rows=178142.00 loops=1)
                     Index Cond: (book_date > '2017-07-15 15:00:00+00'::timestamp with time zone)
                     Index Searches: 1
                     Buffers: shared read=283
 Planning:
   Buffers: shared hit=35 read=6
 Planning Time: 1.428 ms
 Execution Time: 15.853 ms
(20 rows)
```

Строка `Heap Blocks: exact` говорит о том, что все фрагменты битовой карты построены с точностью до строк, перепроверка не выполняется.

Уменьшим размер выделяемой памяти:
```sql
SET work_mem = '64kB';

SET
```

Повторим запрос:
```sql
\bind '2017-07-15 15:00:00+00'
```

```sql
EXPLAIN (analyze, costs off, timing off)
SELECT count(*) FROM bookings WHERE total_amount < 5000 AND book_date > $1;

                                                          QUERY PLAN                                                           
-------------------------------------------------------------------------------------------------------------------------------
 Aggregate (actual rows=1.00 loops=1)
   Buffers: shared hit=464 read=1205
   ->  Bitmap Heap Scan on bookings (actual rows=129.00 loops=1)
         Recheck Cond: ((total_amount < '5000'::numeric) AND (book_date > '2017-07-15 15:00:00+00'::timestamp with time zone))
         Rows Removed by Index Recheck: 89895
         Heap Blocks: exact=811 lossy=568
         Buffers: shared hit=464 read=1205
         ->  BitmapAnd (actual rows=0.00 loops=1)
               Buffers: shared hit=290
               ->  Bitmap Index Scan on bookings_total_amount_idx (actual rows=1471.00 loops=1)
                     Index Cond: (total_amount < '5000'::numeric)
                     Index Searches: 1
                     Buffers: shared hit=7
               ->  Bitmap Index Scan on bookings_book_date_idx (actual rows=178142.00 loops=1)
                     Index Cond: (book_date > '2017-07-15 15:00:00+00'::timestamp with time zone)
                     Index Searches: 1
                     Buffers: shared hit=283
 Planning:
   Buffers: shared hit=4
 Planning Time: 0.302 ms
 Execution Time: 104.523 ms
(21 rows)
```

- в узле `Heap Blocks` появились `lossy` фрагменты битовой карты - с точностью до страниц
- `Rows Removed by Index Recheck` со значением `89895` - количество строк, которое не прошло перепроверку

Восстановим значение параметра:
```sql
RESET work_mem;

RESET
```


## Кластеризация

Если данные в таблице физически упорядоченны, обычное индексное сканирование не будет читать табличную страницу повторно.
В таком нечестом на практике случае метод сканирования по битовой карте теряет смысл, проигрывая обычному индексному сканированию.

Сейчас строки таблицы физически упорядоченны по номеру бронирования:
```sql
SELECT * FROM bookings LIMIT 10;

 book_ref |       book_date        | total_amount 
----------+------------------------+--------------
 000004   | 2016-08-13 12:40:00+00 |     55800.00
 00000F   | 2017-07-05 00:12:00+00 |    265700.00
 000010   | 2017-01-08 16:45:00+00 |     50900.00
 000012   | 2017-07-14 06:02:00+00 |     37900.00
 000026   | 2016-08-30 08:08:00+00 |     95600.00
 00002D   | 2017-05-20 15:45:00+00 |    114700.00
 000034   | 2016-08-08 02:46:00+00 |     49100.00
 00003F   | 2016-12-12 12:02:00+00 |    109800.00
 000048   | 2016-09-16 22:57:00+00 |     92400.00
 00004A   | 2016-10-13 18:57:00+00 |     29000.00
(10 rows)
```

Переупорядочим строки в соответствии с индексом по столбцу `total_amount`:
```sql
CLUSTER bookings USING bookings_total_amount_idx;

CLUSTER
```

Команда `CLUSTER` устанавливает исключительную блокировку, поскольку полностью перестраивает таблицу.
Строки упорядочиваются, но не поддерживаются в упорядоченном виде - в процессе работы кластеризация будет деградировать.

```sql
VACUUM ANALYZE bookings;

VACUUM
```

Убедимся, что строки упорядоченны по стоимости:
```sql
SELECT * FROM bookings LIMIT 10;

 book_ref |       book_date        | total_amount 
----------+------------------------+--------------
 00F39E   | 2017-02-12 01:11:00+00 |      3400.00
 0103E1   | 2017-04-03 06:32:00+00 |      3400.00
 013695   | 2016-11-01 06:30:00+00 |      3400.00
 0158C0   | 2017-04-24 04:47:00+00 |      3400.00
 01AD57   | 2017-03-15 13:11:00+00 |      3400.00
 020E97   | 2017-01-02 09:25:00+00 |      3400.00
 021FD3   | 2017-05-27 07:20:00+00 |      3400.00
 0278C7   | 2017-02-04 14:42:00+00 |      3400.00
 029452   | 2016-12-10 10:51:00+00 |      3400.00
 0355DD   | 2016-09-19 10:22:00+00 |      3400.00
(10 rows)
```

До кластеризации использовалось сканирование по битовой карте, но теперь проще и выгоднее сделать обычное индексное сканирование:
```sql
EXPLAIN (costs off) SELECT * FROM bookings WHERE total_amount < 5000;

                       QUERY PLAN                       
--------------------------------------------------------
 Index Scan using bookings_total_amount_idx on bookings
   Index Cond: (total_amount < '5000'::numeric)
(2 rows)
```