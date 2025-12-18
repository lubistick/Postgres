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

### Чтение индексных страниц

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
`Buffers` - количество индексных страниц, которые потребовалось прочитать.

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


## Сканирование только индекса

Если вся необходимая информация содержится в самом индексе, то нет необходимости обращаться к таблице - за исключением проверки видимости:
```sql
EXPLAIN SELECT book_ref FROM bookings WHERE book_ref <= '100000';

                                        QUERY PLAN                                         
-------------------------------------------------------------------------------------------
 Index Only Scan using bookings_pkey on bookings  (cost=0.43..4180.24 rows=146732 width=7)
   Index Cond: (book_ref <= '100000'::bpchar)
(2 rows)
```
`Index Only Scan` - сканирование только индекса.

### Карта видимости

В каком состоянии у нас была карта видимости?
Действительно ли при выборе такого метода доступа мы не обращаемся к табличным строкам?
Узнать это можно только РЕАЛЬНО выполнив запрос.

Посмотрим план с помощью `EXPLAIN ANALYZE`. Этот вариант команды не просто показывает план,
но и реально выполняет запрос, и вывод содержит больше полезных сведений:
```sql
EXPLAIN (analyze, costs off, timing off)
SELECT book_ref FROM bookings WHERE book_ref <= '100000';

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
- `Planning Time` - сколько времени ушло на построение плана
- `Execution Time` - сколько времени ушло на выполнение запроса

Таким образом мы можем сравнить, о чем планировщик думал, когда строил план, и что реально получилось в процессе выполнения.

В нашем случае не пришлось обращаться к таблице, потому что карта видимости актуальна.

Обновим первую строку таблицы:
```sql
UPDATE bookings SET total_amount = total_amount WHERE book_ref = '000004';

UPDATE 1
```

Сколько версий строк придется проверить теперь?
```sql
EXPLAIN (analyze, costs off, timing off)
SELECT book_ref FROM bookings WHERE book_ref <= '100000';

                                  QUERY PLAN                                  
------------------------------------------------------------------------------
 Index Only Scan using bookings_pkey on bookings (actual rows=132109 loops=1)
   Index Cond: (book_ref <= '100000'::bpchar)
   Heap Fetches: 158
 Planning Time: 0.072 ms
 Execution Time: 20.291 ms
(5 rows)
```

Используется `Index Only Scan`, однако в таблицу мы ходили.
Было прочитано уже `158` версий строк с измененной страницы, потому что карта видимости еще не обновилась.
Приходится проверять все версии строк, попадающие на измененную страницу.


## Параллельное индексное сканирование

Найдем сумму одной четверти всех бронирований:
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

### Оценка стоимости

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

Во-вторых, стоимость индексного доступа. Она не делится между процессами, т.к. индекс читается процессами последовательно, страница за страницей:
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


## Параллельное сканирование только индекса

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


## Многоколоночный индекс

Индекс можно создать по нескольким столбцам.
В этом случае имеет значение порядок следования столбцов и порядок сортировки.
Такой индекс ускоряет поиск, если в запросе есть условие на один или несколько первых ключей, поскольку в индексных записях ключи отсортированы сначала по первому столбцу, затем по второму и т.д.

На таблице перелетов создан многоколоночный индекс по столбцам `ticket_no` и `flight_id`.
Запрос с условием по обоим колонкам использует индекс:
```sql
EXPLAIN SELECT * FROM ticket_flights WHERE ticket_no = '0005432000284' AND flight_id  = 187662;

                                        QUERY PLAN                                         
-------------------------------------------------------------------------------------------
 Index Scan using ticket_flights_pkey on ticket_flights  (cost=0.56..8.58 rows=1 width=32)
   Index Cond: ((ticket_no = '0005432000284'::bpchar) AND (flight_id = 187662))
(2 rows)
```

Запрос с условием по номеру билета - тоже:
```sql
EXPLAIN SELECT * FROM ticket_flights WHERE ticket_no = '0005432000284';

                                         QUERY PLAN                                         
--------------------------------------------------------------------------------------------
 Index Scan using ticket_flights_pkey on ticket_flights  (cost=0.56..16.58 rows=3 width=32)
   Index Cond: (ticket_no = '0005432000284'::bpchar)
(2 rows)
```

Однако если запрос содержит условие только на второй столбец, индекс оказывается бесполезным:
```sql
EXPLAIN SELECT * FROM ticket_flights WHERE flight_id  = 187662;

                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Gather  (cost=1000.00..114685.10 rows=103 width=32)
   Workers Planned: 2
   ->  Parallel Seq Scan on ticket_flights  (cost=0.00..113674.80 rows=43 width=32)
         Filter: (flight_id = 187662)
 JIT:
   Functions: 2
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(7 rows)
```


## Include-индексы

Неключевые столбцы include-индекса:
- не используются при поиске по индексу
- не учитываются ограничением уникальности
- значения хранятся в индексной записи и возвращаются без обращения к таблице

Индекс `tickets_pkey` не является покрывающим для приведенного запроса,
поскольку в нем требуется не только столбец `ticket_no` (есть в индексе), но и `book_ref`:
```sql
EXPLAIN (analyze, buffers, costs off, summary off)
SELECT ticket_no, book_ref FROM tickets WHERE ticket_no = '0005435990286';

                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Index Scan using tickets_pkey on tickets (actual time=1.151..1.152 rows=1 loops=1)
   Index Cond: (ticket_no = '0005435990286'::bpchar)
   Buffers: shared read=4
 Planning:
   Buffers: shared hit=20 read=2
(5 rows)
```

Создадим include-индекс, добавив в него неключевой столбец `book_ref`, так как он требуется запросу:
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
Теперь оптимизатор выбирает метод `Index Only Scan` и использует только что созданный индекс.
Поскольку карта видимости актуальна, обращаться к таблице не пришлось (значение `Heap Fetches` равно `0`).

В include-индекс можно включать столбцы с типами данных, которые не поддерживаются B-деревом, например, геометрические типы и xml.


## Исключение дубликатов в индексе

Сравним размер индекса без исключения дубликатов и с исключением.
Пример подобран таким образом, что в индексе будет много повторяющихся значений.

Создадим индекс, отключив исключение дубликатов с помощью параметра хранения `deduplicate_items`:
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

Выполним запрос с помощью индекса, отключив для этого последовательное сканирование:
```sql
SET enable_seqscan = off;

SET
```

```sql
EXPLAIN (analyze, buffers, costs off)
SELECT fare_conditions FROM ticket_flights;

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
Пришлось прочитать `23876` индексных страниц (значение `Buffers`).

Удалим индекс и создадим его заново.
Параметр хранения `deduplicate_items` не указываем, т.к. по умолчанию он включен:
```sql
DROP INDEX dedup_test;

DROP INDEX
```

```sql
CREATE INDEX dedup_test ON ticket_flights (fare_conditions);

CREATE INDEX
```

Посмотрим размер индекса:
```sql
SELECT pg_size_pretty(pg_total_relation_size('dedup_test'))

 pg_size_pretty 
----------------
 56 MB
(1 row)
```
Размер сократился более чем в `3` раза.

Повторно выполним запрос:
```sql
EXPLAIN (analyze, buffers, costs off)
SELECT fare_conditions FROM ticket_flights;

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

Количество `Buffers` тоже сократилось примерно в `3` раза. И запрос стал выполняться быстрее.

Включим последовательное сканирование:
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

Чтобы посмотреть на план такого запроса, запретим индексный доступ:
```sql
SET enable_indexscan = off;
    
SET
```

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

Чтобы выполнить сортировку, надо получить данные полностью.
Это неэффективно, если в результате требуется только часть выборки:
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
EXPLAIN (analyze, timing off)
SELECT * FROM flights ORDER BY flight_id LIMIT 100;

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

Алгоритм метода сортировки `top-N heapsort` следующий - 100 раз ходим за самым маленьким значением в таблицу.
Этот способ позволяет отдавать строки одну за другой и остановиться после того, как эти 100 строк были выбраны.

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
общая стоимость плана (`4.26`) меньше, потому что узел `Index Scan` сразу отдает строки узлу `Limit` (строки упорядочены).

## Максимальная сумма бронирования

Напишем запрос, выбирающий максимальную сумму бронирования через предложения `ORDER BY` и `LIMIT`:
```sql
EXPLAIN SELECT total_amount FROM bookings ORDER BY total_amount DESC LIMIT 1;

                                         QUERY PLAN                                         
--------------------------------------------------------------------------------------------
 Limit  (cost=27641.46..27641.58 rows=1 width=6)
   ->  Gather Merge  (cost=27641.46..232902.56 rows=1759258 width=6)
         Workers Planned: 2
         ->  Sort  (cost=26641.44..28840.51 rows=879629 width=6)
               Sort Key: total_amount DESC
               ->  Parallel Seq Scan on bookings  (cost=0.00..22243.29 rows=879629 width=6)
(6 rows)
```
Планировщик последовательно сканирует всю таблицу, а затем сортирует данные.
Это неэффективный доступ, поскольку нам требуется только одно значение.
Но, пока нет индекса, это единственно возможный способ.

Можно сформировать запрос с помощью агрегатной функции `max()`:
```sql
EXPLAIN SELECT max(total_amount) FROM bookings;

                                         QUERY PLAN                                         
--------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=25442.58..25442.59 rows=1 width=32)
   ->  Gather  (cost=25442.36..25442.57 rows=2 width=32)
         Workers Planned: 2
         ->  Partial Aggregate  (cost=24442.36..24442.37 rows=1 width=32)
               ->  Parallel Seq Scan on bookings  (cost=0.00..22243.29 rows=879629 width=6)
(5 rows)
```

### Оптимизация

Создадим индекс:
```sql
CREATE INDEX ON bookings(total_amount);

CREATE INDEX
```

Повторяем запрос с `ORDER BY` и `LIMIT`:
```sql
EXPLAIN SELECT total_amount FROM bookings ORDER BY total_amount DESC LIMIT 1;

                                                       QUERY PLAN                                                       
------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.43..0.46 rows=1 width=6)
   ->  Index Only Scan Backward using bookings_total_amount_idx on bookings  (cost=0.43..54879.08 rows=2111110 width=6)
(2 rows)
```
Теперь планировщик выбрал сканирование только индекса.

Повторяем запрос через `max()`:
```sql
EXPLAIN SELECT max(total_amount) FROM bookings;

                                                           QUERY PLAN                                                           
--------------------------------------------------------------------------------------------------------------------------------
 Result  (cost=0.46..0.47 rows=1 width=32)
   InitPlan 1 (returns $0)
     ->  Limit  (cost=0.43..0.46 rows=1 width=6)
           ->  Index Only Scan Backward using bookings_total_amount_idx on bookings  (cost=0.43..60156.85 rows=2111110 width=6)
                 Index Cond: (total_amount IS NOT NULL)
(5 rows)
```
И в этом случае планировщик выбрал сканирование только индекса,
добавив условие `total_amount IS NOT NULL`, поскольку функция `max()` не должна учитывать неопределенные значения.

`InitPlan` - узел плана, соответствует подзапросу, который выполняется один раз.

Фактически планировщик переформулировал запрос следующим образом:
```sql
EXPLAIN SELECT (
  SELECT total_amount FROM bookings
  WHERE total_amount IS NOT NULL
  ORDER BY total_amount DESC LIMIT 1
);

                                                           QUERY PLAN                                                           
--------------------------------------------------------------------------------------------------------------------------------
 Result  (cost=0.46..0.47 rows=1 width=16)
   InitPlan 1 (returns $0)
     ->  Limit  (cost=0.43..0.46 rows=1 width=6)
           ->  Index Only Scan Backward using bookings_total_amount_idx on bookings  (cost=0.43..60156.85 rows=2111110 width=6)
                 Index Cond: (total_amount IS NOT NULL)
(5 rows)
```
Если бы запрос выполнялся несколько раз, вместо `InitPlan` был бы узел `SubPlan`.


## Порядок сортировки при создании индекса

Порядок важен для многоколоночных индексов.
Создадим индекс на таблице рейсов по аэропортам вылета и прилета:
```sql
CREATE INDEX dep_arr ON flights(departure_airport, arrival_airport);

CREATE INDEX
```

Индекс используется при сортировке в одном направлении:
```sql
EXPLAIN SELECT * FROM flights ORDER BY departure_airport, arrival_airport;

                                   QUERY PLAN                                    
---------------------------------------------------------------------------------
 Index Scan using dep_arr on flights  (cost=0.29..14414.16 rows=214867 width=63)
(1 row)
```

И в одном направлении, но в обратном порядке:
```sql
EXPLAIN SELECT * FROM flights ORDER BY departure_airport DESC, arrival_airport DESC;

                                        QUERY PLAN                                        
------------------------------------------------------------------------------------------
 Index Scan Backward using dep_arr on flights  (cost=0.29..14414.16 rows=214867 width=63)
(1 row)
```

Но не при сортировке в разных направлениях:
```sql
EXPLAIN SELECT * FROM flights ORDER BY departure_airport, arrival_airport DESC;

                              QUERY PLAN                              
----------------------------------------------------------------------
 Sort  (cost=31883.96..32421.12 rows=214867 width=63)
   Sort Key: departure_airport, arrival_airport DESC
   ->  Seq Scan on flights  (cost=0.00..4772.67 rows=214867 width=63)
(3 rows)
```

Здесь приходится отдельно выполнять сортировку результатов.

В этом случае поможет другой индекс:
```sql
CREATE INDEX dep_asc_arr_desc ON flights(departure_airport, arrival_airport DESC);

CREATE INDEX
```

```sql
EXPLAIN SELECT * FROM flights ORDER BY departure_airport, arrival_airport DESC;

                                        QUERY PLAN                                        
------------------------------------------------------------------------------------------
 Index Scan using dep_asc_arr_desc on flights  (cost=0.29..14414.16 rows=214867 width=63)
(1 row)
```

А также в этом:
```sql
EXPLAIN SELECT * FROM flights ORDER BY departure_airport DESC, arrival_airport;

                                            QUERY PLAN                                             
---------------------------------------------------------------------------------------------------
 Index Scan Backward using dep_asc_arr_desc on flights  (cost=0.29..14414.16 rows=214867 width=63)
(1 row)
```
