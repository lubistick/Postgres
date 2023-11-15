# Индексное сканирование

Индекс - вспомогательная структура во внешней памяти, сопоставляет ключи и идентификаторы строк таблицы.

Индексы:
- ускоряют доступ при высокой селективности (когда нужно выбрать маленький объем данных)
- поддерживают ограничения целостности
- позволяют получить отсортированные данные (индекс поддерживает те типы данных, которые можно сортировать операциями "больше", "меньше")

## B-дерево

Столбец `book_ref` в таблице бронирований является первичным ключом.
Для него автоматически был создан индекс `bookings_pkey`:
```sql
\d bookings

                        Table "bookings.bookings"
    Column    |           Type           | Collation | Nullable | Default
--------------+--------------------------+-----------+----------+---------
 book_ref     | character(6)             |           | not null |
 book_date    | timestamp with time zone |           | not null |
 total_amount | numeric(10,2)            |           | not null |
Indexes:
    "bookings_pkey" PRIMARY KEY, btree (book_ref)
Referenced by:
    TABLE "tickets" CONSTRAINT "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
```
B-дерево или `btree` индекс:
- это сбалансированное дерево, т.е. от корневого узла до листовой страницы всегда одинаковое расстояние
- листовые страницы упорядочены и связаны между собой двунаправленным списком
- для больших таблиц такой индекс будет сильно ветвистым

## Поиск одного значения по индексу

Проверим план запроса:
```sql
EXPLAIN SELECT * FROM bookings WHERE book_ref = 'CDE08B';

                                  QUERY PLAN                                   
-------------------------------------------------------------------------------
 Index Scan using bookings_pkey on bookings  (cost=0.43..8.45 rows=1 width=21)
   Index Cond: (book_ref = 'CDE08B'::bpchar)
(2 rows)
```
План:
- `Index Scan` - доступ по индексу, указано имя использованного индекса
- `Index Cond` - условие доступа

### Оценка стоимости

Начальная стоимость `0.43` - оценка ресурсов для спуска к листовому узлу B-дерева.
Она зависит:
- от логарифма количества листовых узлов (количества операций сравнения, которые надо выполнить)
- от высоты дерева (спускаемся от корня индекса до листовой странички, после этого мы получим ссылку на табличную строку и будем готовы ее извлекать)

При оценке считается, что необходимые страницы окажутся в кэше и оцениваются только ресурсы процессора.
Цифра получается небольшой.

Полная стоимость `8.45` - оценка чтения необходимых листовых страниц индекса и табличных страниц.
Один узел `Index Scan` подразумевает обращение и к индексу, и к таблице.
Поскольку индекс уникальный, будет прочитана одна индексная страница и одна табличная.
Каждое из чтений оценивается параметром:
```sql
SELECT current_setting('random_page_cost');

 current_setting 
-----------------
 4               
(1 row)
```
Итого получаем `8`, и еще немного добавляет оценка процессорного времени на обработку строк.

Значение `random_page_cost` по умолчанию больше, чем `seq_page_cost`, поскольку произвольный доступ стоит дороже.
Хотя для SSD-дисков этот параметр следует существенно уменьшать (до `2`, до `1.5` или даже до `1.1`).


### Дополнительные условия

Выполним запрос:
```sql
EXPLAIN SELECT * FROM bookings WHERE book_ref = 'CDE08B' AND total_amount > 1000;

                                  QUERY PLAN                                   
-------------------------------------------------------------------------------
 Index Scan using bookings_pkey on bookings  (cost=0.43..8.45 rows=1 width=21) 
   Index Cond: (book_ref = 'CDE08B'::bpchar)                                   
   Filter: (total_amount > '1000'::numeric)                                    
(3 rows)
```
`Filter` - дополнительные условия, которые можно проверить только по таблице.


## Поиск по диапазону

Чтобы получить данные из индекса, нужно спуститься от корня дерева к левому листовому узлу и пройти по списку листовых страниц.
Поэтому индексное сканирование всегда возвращает данные в том порядке, в котором они хранятся в девере индекса (порядок был указан при создании индекса):
```sql
EXPLAIN (costs off) SELECT * FROM bookings WHERE book_ref > '000900' AND book_ref < '000939' ORDER BY book_ref;

                                   QUERY PLAN                                    
---------------------------------------------------------------------------------
 Index Scan using bookings_pkey on bookings
   Index Cond: ((book_ref > '000900'::bpchar) AND (book_ref < '000939'::bpchar))
(2 rows)
```
Обратим внимание, что нет узла сортировки.

Тот же самый индекс может использоваться и для получения строк в обратном порядке:
```sql
EXPLAIN (costs off) SELECT * FROM bookings WHERE book_ref > '000900' AND book_ref < '000939' ORDER BY book_ref DESC;

                                   QUERY PLAN                                    
---------------------------------------------------------------------------------
 Index Scan Backward using bookings_pkey on bookings
   Index Cond: ((book_ref > '000900'::bpchar) AND (book_ref < '000939'::bpchar))
(2 rows)
```
План такой же, только добавилось ключевое слово `Backward`.
В этом случае нужно спуститься по дереву от корневой страницы к другому (правому) краю диапазона и произвести поиск в обратном направлении
(листовые страницы связаны двунаправленным списком).

### Чтение страниц

Выполним запрос:
```sql
EXPLAIN (analyze, buffers, costs off, timing off, summary off)
SELECT * FROM bookings
WHERE book_ref > '000900' AND book_ref < '000939'
ORDER BY book_ref DESC;

                                   QUERY PLAN                                    
---------------------------------------------------------------------------------
 Index Scan Backward using bookings_pkey on bookings (actual rows=5 loops=1)
   Index Cond: ((book_ref > '000900'::bpchar) AND (book_ref < '000939'::bpchar))
   Buffers: shared hit=4 read=1
 Planning:
   Buffers: shared hit=8
(5 rows)
```
`Buffers` - количество страниц, которые потребовалось прочитать.

Сравним поиск по диапазону с повторяющимся поиском отдельных значений.
Получим тот же результат с помощью конструкции `IN`:
```sql
EXPLAIN (analyze, buffers, costs off)
SELECT * FROM bookings
WHERE book_ref IN ('000906', '000909', '000917', '000930', '000938')
ORDER BY book_ref DESC;

                                          QUERY PLAN                                           
-----------------------------------------------------------------------------------------------
 Index Scan Backward using bookings_pkey on bookings (actual time=0.115..0.129 rows=5 loops=1)
   Index Cond: (book_ref = ANY ('{000906,000909,000917,000930,000938}'::bpchar[]))
   Buffers: shared hit=24
 Planning Time: 0.061 ms
 Execution Time: 0.145 ms
(5 rows)
```
Количество страниц увеличилось до `24`, поскольку в этом случае приходится спускаться от корня к каждому значению.










## Параллельное индексное сканирование

Индексное сканирование может выполняться в параллельном режиме.
В качестве примера найдем сумму одной четверти всех бронирований:

```sql
EXPLAIN SELECT sum(total_amount) FROM bookings WHERE book_ref < '400000';

                                                    QUERY PLAN                                                    
------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=16861.86..16861.87 rows=1 width=32)
   ->  Gather  (cost=16861.64..16861.85 rows=2 width=32)
         Workers Planned: 2
         ->  Partial Aggregate  (cost=15861.64..15861.65 rows=1 width=32)
               ->  Parallel Index Scan using bookings_pkey on bookings  (cost=0.43..15312.03 rows=219843 width=6)
                     Index Cond: (book_ref < '400000'::bpchar)
(6 rows)
```
Аналогичный план мы уже видели при параллельном последовательном сканировании,
но в данном случае данные читаются с помощью индекса - узел `Parallel Index Scan`.

Стоимость обусловлена двумя составляющими.
Во-первых, стоимость доступа к `1/4` табличных страниц, поделенных между процессами - аналогично последовательному сканированию:
```sql
SELECT
  round(
    (relpages / 4.0) * current_setting('seq_page_cost')::real +
    (reltuples / 4.0) / 2.4 * current_setting('cpu_tuple_cost')::real
  )
FROM pg_class WHERE relname = 'bookings';

 round 
-------
  5561
(1 row)
```

Вторая составляющая - индексный доступ (не делится между процессами, т.к. индекс читается процессами последовательно, страница за страницей):
```sql
SELECT
  round(
    (relpages / 4.0) * current_setting('random_page_cost')::real +
    (reltuples / 4.0) * current_setting('cpu_index_tuple_cost')::real +
    (reltuples / 4.0) * current_setting('cpu_operator_cost')::real
  )
FROM pg_class WHERE relname = 'bookings_pkey';

 round 
-------
  9750
(1 row)
```
Стоимость `cpu_operator_cost` учитывает необходимость сравнения значений (операция "меньше либо равно").

Сложим `9750` и `5561` - получим `15311` (полная стоимость `Parallel Index Scan`).


## Исключительно индексное сканирование

Если вся необходимая информация содержится в самом индексе, то нет необходимости обращаться к таблице - за исключением проверки видимости:
```sql
EXPLAIN SELECT book_ref FROM bookings WHERE book_ref <= '100000';

                                        QUERY PLAN                                         
-------------------------------------------------------------------------------------------
 Index Only Scan using bookings_pkey on bookings  (cost=0.43..4180.24 rows=146732 width=7)
   Index Cond: (book_ref <= '100000'::bpchar)
(2 rows)


```
Вопрос в каком состоянии у нас была карта видимости и действительно ли при выборе такого метода доступа мы не обращаемся к табличным строкам.

Узнать это можно только выполнив запрос.

Посмотрим план с помощью EXPLAIN ANALYZE. Этот вариант команды EXPLAIN не просто показывает план,
но и реально выполняет запрос, и вывод содержит больше полезных сведений.
```sql
EXPLAIN (ANALYZE, COSTS OFF, TIMING OFF) SELECT book_ref FROM bookings WHERE book_ref <= '100000';

                                  QUERY PLAN                                  
------------------------------------------------------------------------------
 Index Only Scan using bookings_pkey on bookings (actual rows=132109 loops=1)
   Index Cond: (book_ref <= '100000'::bpchar)
   Heap Fetches: 0
 Planning Time: 0.046 ms
 Execution Time: 28.348 ms
(5 rows)
```
`EXPLAIN ANALYZE` для каждого узла в плане после слова `actual` показывает:
- `rows` - сколько реально строк было выбрано `132109`
- `loops` - сколько раз выполнялся узел плана `1`

Дополнительные параметры в плане:
- `Heap Fetches` - сколько строк было проверено с помощью таблицы `0`
- `Planning Time` - сколько времени ушло на построение плана (число разное от раза к разу)
- `Execution Time` - сколько времени ушло на выполнение запроса (число разное от раза к разу)



Таким образом мы сможем сравнивать, о чем планировщик думал, когда строил план, и что реально получилось в процессе выполнения.

В нашем случае не пришлось обращаться к таблице, потому что очистка таблицы выполнялась, следовательно, и карта видимости обновлена:
```sql
SELECT vacuum_count, autovacuum_count FROM pg_stat_all_tables WHERE relname = 'bookings';

 vacuum_count | autovacuum_count 
--------------+------------------
            0 |                1
(1 row)
```

Выполним очистку таблицы:
```sql
VACUUM bookings;

VACUUM
```
Вакуум не помогает в обновлении карты видимости! (исправить текст выше)


Обновим первую строку таблицы:
```sql
UPDATE bookings SET total_amount = total_amount WHERE book_ref = '000004';

UPDATE 1
```

Сколько версий строк придется проверить теперь?
```sql
EXPLAIN (ANALYZE, COSTS OFF, TIMING OFF) SELECT book_ref FROM bookings WHERE book_ref <= '100000';

                                  QUERY PLAN                                  
------------------------------------------------------------------------------
 Index Only Scan using bookings_pkey on bookings (actual rows=132109 loops=1)
   Index Cond: (book_ref <= '100000'::bpchar)
   Heap Fetches: 158
 Planning Time: 0.072 ms
 Execution Time: 20.291 ms
(5 rows)
```
Проверять приходится все версии, попадающие на измененную страницу.
Heap Fetches: 158 - в данном случае было прочитано уже 158 версий строк с той нашей измененной страницы.
Потому что карта видимости еще не обновилась.
Используется Index Only Scan, однако в таблицу мы ходили.


## Include индексы

Еще один вариант создать покрывающий индекс - создать include индекс.

Неключевые столбцы:
- не используются при поиске по индексу
- не учитываются ограничением уникальности
- значения хранятся в индексной записи и возвращаются без обращения к таблице

Индекс tickets_pkey не является покрывающим для приведенного запроса,
поскольку в нем требуется не только столбец ticket_no (есть в индексе), но и book_ref:
```sql
EXPLAIN (analyze, buffers, costs off, summary off) SELECT ticket_no, book_ref FROM tickets WHERE ticket_no = '0005435990286';

                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Index Scan using tickets_pkey on tickets (actual time=1.151..1.152 rows=1 loops=1)
   Index Cond: (ticket_no = '0005435990286'::bpchar)
   Buffers: shared read=4
 Planning:
   Buffers: shared hit=20 read=2
(5 rows)
```

Создадим include-индекс, добавив в него неключевой столбец book_ref, так как он требуется запросу:
```sql
CREATE UNIQUE INDEX ON tickets (ticket_no) INCLUDE (book_ref);

CREATE INDEX
```

Повторим запрос:
```sql
EXPLAIN (analyze, buffers, costs off, summary off)
SELECT ticket_no, book_ref FROM tickets WHERE ticket_no = '0005435990286';

                                                QUERY PLAN                                                 
-----------------------------------------------------------------------------------------------------------
 Index Only Scan using tickets_ticket_no_book_ref_idx on tickets (actual time=0.049..0.050 rows=1 loops=1)
   Index Cond: (ticket_no = '0005435990286'::bpchar)
   Heap Fetches: 0
   Buffers: shared hit=1 read=3
 Planning:
   Buffers: shared hit=13 read=1
(6 rows)
```
Теперь оптимизатор выбирает метод Index Only Scan и использует только что созданный индекс.
Количество прочитанных страниц сократилось (в моем примере нет).
Поскольку карта видимости актуальна, обращаться к таблице не пришлось (Heap Fetches: 0).

В include-индекс можно включать столбцы с типами данных, которые не поддерживаются B-деревом, например, геометрические типы и xml.



## Параллельное исключительно индексное сканирование

(параллельное сканирование только индекса)

Выполним запрос:
```sql
EXPLAIN SELECT count(book_ref) FROM bookings WHERE book_ref <= '400000';

                                                      QUERY PLAN                                                       
-----------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=13499.01..13499.02 rows=1 width=8)
   ->  Gather  (cost=13498.80..13499.01 rows=2 width=8)
         Workers Planned: 2
         ->  Partial Aggregate  (cost=12498.80..12498.81 rows=1 width=8)
               ->  Parallel Index Only Scan using bookings_pkey on bookings  (cost=0.43..11949.09 rows=219881 width=7)
                     Index Cond: (book_ref <= '400000'::bpchar)
(6 rows)
```

Стоимость доступа к таблице здесь учитывает только обработку строк, без ввода-вывода:
```sql
SELECT
  round(
    (reltuples / 4.0) / 2.4 * current_setting('cpu_tuple_cost')::real
  )
FROM pg_class WHERE relname = 'bookings';

 round 
-------
  2199
(1 row)
```
Вклад индексного доступа остается прежним - `9750`. Добавим `2199` и получим `11949` (полная стоимость узла `Parallel Index Only Scan`).

Дальше мы не будем подробно останавливаться на расчете стоимости.
Для этого используется достаточно сложная математическая модель.
В нашу задачу входит только показать общий принцип.


## Исключение дубликатов в индексе

Сравним размер индекса без исключения дубликатов и с исключением.
Пример подобран таким образом, что в индексе должно быть много повторяющихся значений.

Сначала создадим индекс, отключив исключение дубликатов с помощью параметра хранения deduplicate_items:
```sql
CREATE INDEX dedup_test ON ticket_flights (fare_conditions) WITH (deduplicate_items = off);

CREATE INDEX
```

Посмотрим размер созданного индекса:
```sql
SELECT pg_size_pretty(pg_total_relation_size('dedup_test'));

 pg_size_pretty 
----------------
 187 MB
(1 row)
```

Выполним запрос с помощью индекса, отключив для этого последовательное сканирование.
Обратите внимание на значение Buffers и время выполнения запроса:
```sql
SET enable_seqscan = off;

SET
```

```sql
EXPLAIN (analyze, buffers, costs off) SELECT fare_conditions FROM ticket_flights;

                                              QUERY PLAN                                               
-------------------------------------------------------------------------------------------------------
 Index Only Scan using dedup_test on ticket_flights (actual time=0.569..1345.772 rows=8391852 loops=1)
   Heap Fetches: 0
   Buffers: shared hit=5 read=23876
 Planning:
   Buffers: shared hit=19 read=1 dirtied=1
 Planning Time: 0.476 ms
 Execution Time: 1671.016 ms
(7 rows)
```

Удалим индекс и создадим его заново. Параметр хранения deduplicate_items не указываем, т.к. по умолчанию он включен:
```sql
DROP INDEX dedup_test;

DROP INDEX
```

```sql
CREATE INDEX dedup_test ON ticket_flights (fare_conditions);

CREATE INDEX
```

Опять посмотрим размер индекса:
```sql
SELECT pg_size_pretty(pg_total_relation_size('dedup_test'))

 pg_size_pretty 
----------------
 56 MB
(1 row)
```
Размер сократился более чем в 3 раза.

Повторно выполним запрос:
```sql
EXPLAIN (analyze, buffers, costs off) SELECT fare_conditions FROM ticket_flights;

                                              QUERY PLAN                                              
------------------------------------------------------------------------------------------------------
 Index Only Scan using dedup_test on ticket_flights (actual time=0.041..534.826 rows=8391852 loops=1)
   Heap Fetches: 0
   Buffers: shared hit=5 read=7077
 Planning:
   Buffers: shared hit=5 read=1
 Planning Time: 0.269 ms
 Execution Time: 864.220 ms
(7 rows)
```

Количество Buffers тоже сократилось примерно в 3 раза. И запрос стал выполняться быстрее.
```sql
RESET enable_seqscan;

RESET
```

## Сортировка

Есть два способа выполнить сортировку.
Первый - получить строки сканированием подходящего индекса, в этом случае данные автоматически будут отсортированы:

```sql
EXPLAIN SELECT * FROM flights ORDER BY flight_id;

                                     QUERY PLAN                                      
-------------------------------------------------------------------------------------
 Index Scan using flights_pkey on flights  (cost=0.42..8248.95 rows=214867 width=63)
(1 row)
```

Тот же самый индекс может использоваться и для получения строк в обратном порядке:
```sql
EXPLAIN SELECT * FROM flights ORDER BY flight_id DESC;

                                          QUERY PLAN                                          
----------------------------------------------------------------------------------------------
 Index Scan Backward using flights_pkey on flights  (cost=0.42..8248.95 rows=214867 width=63)
(1 row)
```
В этом случае мы спускаемся от корня дерева к правому листовому узлу, и проходим по списку листовых страниц в обратном порядке.

Второй способ - выполнить последовательное сканирование таблицы и затем отсортировать полученные данные.
Чтобы посмотреть на план такого запроса, запретим индексный доступ (аналогично можно запретить и другие методы доступа):
```sql
SET enable_indexscan = off;
    
SET
```
На самом деле это не запрет (объяснить).

```sql
EXPLAIN SELECT * FROM flights ORDER BY flight_id DESC;

                              QUERY PLAN                              
----------------------------------------------------------------------
 Sort  (cost=31883.96..32421.12 rows=214867 width=63)
   Sort Key: flight_id DESC
   ->  Seq Scan on flights  (cost=0.00..4772.67 rows=214867 width=63)
(3 rows)
```

Результаты сканирования передаются узлу `Sort`, который сортирует их по ключу `flight_id`.

Стоимость такого запроса существенно выросла.

Чтобы выполнить сортировку, надо получить весь набор данных полностью.
Это неэффективно, если в результате требуется только часть выборки.
Обратите внимание на стоимость:
```sql
EXPLAIN SELECT * FROM flights ORDER BY flight_id LIMIT 100;

                                        QUERY PLAN                                         
-------------------------------------------------------------------------------------------
 Limit  (cost=9718.54..9730.04 rows=100 width=63)
   ->  Gather Merge  (cost=9718.54..24253.62 rows=126392 width=63)
         Workers Planned: 1
         ->  Sort  (cost=8718.53..9034.51 rows=126392 width=63)
               Sort Key: flight_id
               ->  Parallel Seq Scan on flights  (cost=0.00..3887.92 rows=126392 width=63)
(6 rows)
```

Хотя стоимость и высока, она все же меньше, чем в предыдущем примере. Почему?
```sql
EXPLAIN (ANALYZE, TIMING OFF) SELECT * FROM flights ORDER BY flight_id LIMIT 100;

                                                       QUERY PLAN
------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=9718.54..9730.04 rows=100 width=63) (actual rows=100 loops=1)
   ->  Gather Merge  (cost=9718.54..24253.62 rows=126392 width=63) (actual rows=100 loops=1)
         Workers Planned: 1
         Workers Launched: 1
         ->  Sort  (cost=8718.53..9034.51 rows=126392 width=63) (actual rows=100 loops=2)
               Sort Key: flight_id
               Sort Method: top-N heapsort  Memory: 47kB
               Worker 0:  Sort Method: top-N heapsort  Memory: 39kB
               ->  Parallel Seq Scan on flights  (cost=0.00..3887.92 rows=126392 width=63) (actual rows=107434 loops=2)
 Planning Time: 0.058 ms
 Execution Time: 61.344 ms
(11 rows)
```

Postgres умеет выполнять разные виды сортировок.
`Sort Method` - `top-N heapsort` - который сводится к тому, что мы не всю выборку получаем, а потом ее отсортировываем, а мы просто 100 раз ходим за самым маленьким значением в таблицу.
Этот способ позволяет отдавать строки одну за другой и мы останавливаемся после того, как эти 100 строк были выбраны.

Здесь фактически не сортируется весь набор, а 100 раз находится минимальное значение.

Индексное сканирование позволяет получать данные по мере необходимости. Сравните стоимость с предыдущим вариантом:
```sql
RESET enable_indexscan;
    
RESET
```

```sql
EXPLAIN SELECT * FROM flights ORDER BY flight_id LIMIT 100;

                                        QUERY PLAN                                         
-------------------------------------------------------------------------------------------
 Limit  (cost=0.42..4.26 rows=100 width=63)
   ->  Index Scan using flights_pkey on flights  (cost=0.42..8248.95 rows=214867 width=63)
(2 rows)
```
Несмотря на то, что стоимость всего индексного сканирования (`8248`) достаточно велика,
общая стоимость плана (`4.26`) меньше, потому что мы строчки начинаем получать сразу.

С учетом того, что индекс может отдавать строчки сразу (они упорядочены), то стоимость плана минимальна (`4.26`). 



