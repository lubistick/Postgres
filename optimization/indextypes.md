# Типы индексов


## Класс операторов

Чтобы индексные методы доступа могли работать с разными типами данных, между индексами и типами данных есть посредник - класс операторов.
Классы операторов содаржат операторы и вспомогательные функции чтобы реализовать правила, по которым значения добавляются в индекс и затем ищутся в нем.
Все типы индексов используют классы операторов, даже B-деревья, но они довольно простые - содержат такие операторы как "равно", "больше" и "меньше".


## Хеш индекс

Хеш индекс представляет собой хеш-таблицу, которая хранится на диске.

Идея хеширования состоит в том, что значения любого типа данных равномерно распределяются по ограниченному количеству корзин хеш-таблицы с помощью функции хеширования.
Если хеш-таблица имеет достаточный размер, чтобы в одну корзину в среднем попадало одно значение (хеш-код), поиск значения в хеш-таблице выполняется за константное время.

Хеш-индекс хранит только значения хеш-функции и ссылки на версии строк.
Само индексируемое значение не сохраняется.
Теряется возможность сканирования только индекса, значение можно прочитать только из таблицы.
Размер хеш-индекса увеличивается динамически при добавлении новых значений.
Рост размера происходит скачкообразно - количество корзин удваивается.

Ограничения:
- единственная поддерживаемая операция - поиск по условию равенства
- не поддерживается ограничение уникальности
- нельзя создавать многоколоночный индекс или добавить include столбцы

Посмотрим план запроса:
```sql
EXPLAIN (costs off)
SELECT * FROM seats WHERE seat_no = '31D';

                QUERY PLAN                 
-------------------------------------------
 Seq Scan on seats
   Filter: ((seat_no)::text = '31D'::text)
(2 rows)
```

Поскольку подходящего индекса нет, используется последовательное сканирование.
Создадим хеш-индекс по полю "seat_no":
```sql
CREATE INDEX ON seats USING hash(seat_no);

CREATE INDEX
```

Повторим запрос:
```sql
EXPLAIN (costs off)
SELECT * FROM seats WHERE seat_no = '31D';

                     QUERY PLAN                      
-----------------------------------------------------
 Bitmap Heap Scan on seats
   Recheck Cond: ((seat_no)::text = '31D'::text)
   ->  Bitmap Index Scan on seats_seat_no_idx
         Index Cond: ((seat_no)::text = '31D'::text)
(4 rows)
```

Теперь планировщик использует хеш-индекс и строит битовую карту.
Поменяем условие равенства на "больше":
```sql
EXPLAIN (costs off)
SELECT * FROM seats WHERE seat_no > '31D';

                QUERY PLAN                 
-------------------------------------------
 Seq Scan on seats
   Filter: ((seat_no)::text > '31D'::text)
(2 rows)
```

С неравенствами хеш-индекс использоваться не может.


### Сравнение размера и времени создания индексов B-tree и хеш

Посмотрим структуру таблицы "tickets":
```sql
\d tickets

                        Table "bookings.tickets"
     Column     |         Type          | Collation | Nullable | Default 
----------------+-----------------------+-----------+----------+---------
 ticket_no      | character(13)         |           | not null | 
 book_ref       | character(6)          |           | not null | 
 passenger_id   | character varying(20) |           | not null | 
 passenger_name | text                  |           | not null | 
 contact_data   | jsonb                 |           |          | 
Indexes:
    "tickets_pkey" PRIMARY KEY, btree (ticket_no)
Foreign-key constraints:
    "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
Referenced by:
    TABLE "ticket_flights" CONSTRAINT "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
```

Поле "book_ref" имеет фиксированный размер, а поле "contact_data" имеет тип jsonb.

Включим подсчет времени выполнения запросов:
```sql
\timing on

Timing is on.
```

Создадим хеш-индексы:
```sql
CREATE INDEX tickets_hash_br ON tickets USING hash(book_ref);

CREATE INDEX
Time: 4263.513 ms (00:04.264)
```

```sql
CREATE INDEX tickets_hash_cd ON tickets USING hash(contact_data);

CREATE INDEX
Time: 4969.046 ms (00:04.969)
```

И индексы B-tree:
```sql
CREATE INDEX tickets_btree_br ON tickets(book_ref);

CREATE INDEX
Time: 5804.023 ms (00:05.804)
```

```sql
CREATE INDEX tickets_btree_cd ON tickets(contact_data);

CREATE INDEX
Time: 14214.718 ms (00:14.215)
```

Время создания хеш-индексов примерно одинаково, а время создания btree-индексов зависит от размера индексируемого поля.

```sql
\timing off

Timing is off.
```

Теперь проверим размеры полученных индексов:
```sql
SELECT
  pg_size_pretty(pg_total_relation_size('tickets_hash_br')) AS tickets_hash_br,
  pg_size_pretty(pg_total_relation_size('tickets_hash_cd')) AS tickets_hash_cd,
  pg_size_pretty(pg_total_relation_size('tickets_btree_br')) AS tickets_btree_br,
  pg_size_pretty(pg_total_relation_size('tickets_btree_cd')) AS tickets_btree_cd \gx

-[ RECORD 1 ]----+-------
tickets_hash_br  | 81 MB
tickets_hash_cd  | 80 MB
tickets_btree_br | 59 MB
tickets_btree_cd | 226 MB
```

Размер хеш-индекса не зависит от размера ключа индексирования (индекс не хранит индексируемые значения).
Размер btree-индексов зависит от размера индексируемого поля, (хранит индексируемые значения).

В некоторых случаях хеш-индекс может работать быстрее, чем B-дерево, за счет меньшего размера и фиксированного времени поиска.


## GIST индекс

GiST расшифровывается как generalized search tree - обобщенное дерево поиска.
Это сбалансированное дерево.

Рассмотрим идею такого индекса на примере точек на плоскости.
Плоскость разбивается на несколько прямоугольников, которые в сумме покрывают все точки.
Эти прямоугольники составляют верхний уровень дерева.
На следующем уровне дерева каждый из больших прямоугольников распадается на прямоугольники меньшего размера.
И так далее.
На последнем уровне дерева каждый ограничивающий прямоугольник будет содержать столько точек, сколько помещается на одну индексную страницу.

Такое разбиение позволяет быстро находить точки, лежащие внутри определенной области.
Такой алгоритм индексирования называется R-деревом.

Для демонстрации работы GiST индекса обратимся к таблице "airports_data" (на этой таблице построено представление "airports").
В таблице есть поле "coordinates" типа `point`, по нему и будем строить GiST индекс.

Но сначала выполним следующий запрос - найдем все аэропорты, находящиеся недалеко от Москвы:
```sql
EXPLAIN (costs off) SELECT airport_code FROM airports_data WHERE coordinates <@ '<(37.622513,55.753220),1.0>'::circle;

                          QUERY PLAN                           
---------------------------------------------------------------
 Seq Scan on airports_data
   Filter: (coordinates <@ '<(37.622513,55.75322),1>'::circle)
(2 rows)
```

Без индекса просматривается вся таблица. Создадим GiST-индекс:
```sql
CREATE INDEX airports_gits_idx ON airports_data USING gist(coordinates);

CREATE INDEX
```

Таблица "airposts_data" невелика, поэтому планировщик все равно будет использовать последовательное сканирование.
Временно отключим этот метод доступа:
```sql
SET enable_seqscan = off;

SET
```

Повторим запрос:
```sql
EXPLAIN (costs off) SELECT airport_code FROM airports_data WHERE coordinates <@ '<(37.622513,55.753220),1.0>'::circle;

                            QUERY PLAN                             
-------------------------------------------------------------------
 Index Scan using airports_gits_idx on airports_data
   Index Cond: (coordinates <@ '<(37.622513,55.75322),1>'::circle)
(2 rows)
```

Теперь планировщик получает нужные строки, обращаясь к индексу "airports_gits_idx".

Удалим созданный индекс:
```sql
DROP INDEX airports_gits_idx;

DROP INDEX
```


## SP-GIST индекс

SP-GiST расшифровывается как space partitioning GiST.
Это тоже обобщенное дерево, но области, на которые разбивается плоскость, не пересекаются.
Это несбалансированное дерево.

Рассмотрим пример индекса SP-GiST для точек на плоскости.
Один из вариантов - дерево квадрантов.
Корневой узел разбивает плоскость на четыре части (квадранта) по отношению к выбранной точке (центроиду).
Далее каждый из 4-х квадрантов делится на собственные квадранты.
Разбиение продолжается, пока все точки в квадранте не станут помещаться на одну индексную страницу.

SP-GiST, как и GiST, является каркасом, на основе которого, реализуя классы операторов, можно строить произвольные схемы индексации.
Например, дерево квадрантов для точек реализуется классом операторов `point_ops`.
Другой способ деления плоскости на две части (сначала по горизонтали, потом по вертикали), а не на четыре, называется k-мерным деревом и реализуется классом операторов `kd_point_ops`.

Создадим индекс SP-GiST по полю "coordinates" в таблице "airports_data".
Для точек есть два класса операторов.
По умолчанию используется `point_ops` (дерево квадрантов), мы для примера укажем `kd_point_ops` (k-мерное дерево):
```sql
CREATE INDEX airports_spgist_idx ON airports_data USING spgist(coordinates kd_point_ops);

CREATE INDEX
```

Попробуем найти все аэропорты, расположенные выше (севернее) Надыма:
```sql
EXPLAIN (costs off) SELECT airport_code FROM airports_data WHERE coordinates >^ '(72.69889831542969,65.48090362548828)'::point;

                                     QUERY PLAN                                      
-------------------------------------------------------------------------------------
 Bitmap Heap Scan on airports_data
   Recheck Cond: (coordinates >^ '(72.69889831542969,65.48090362548828)'::point)
   ->  Bitmap Index Scan on airports_spgist_idx
         Index Cond: (coordinates >^ '(72.69889831542969,65.48090362548828)'::point)
(4 rows)
```

Теперь вся таблица сканируется по битовой карте, построенной на основе SP-GiST индекса "airports_spgist_idx".

Удалим индекс:
```sql
DROP INDEX airports_spgist_idx;

DROP INDEX
```


## GIN индекс

GIN - generalized inverted index, обобщенный инвертированный индекс.

Идею этого индексного метода проще всего понять на примере предметного указателя в обычной книге.
На страницах книги встречаются термины, а в предметном указателе перечислены в алфавитном порядке все термины с указанием номеров страниц, на которых они присутствуют.

Индекс GIN в первую очередь используется для индексации документов с целью ускорения полнотекстового поиска.
По сути, это обычное B-дерево, но в него помещены не сами документы, а составляющие их слова.

GIN работает с типами данных, значения которых состоят из элементов, а не являются атомарными.
При этом индексируются не сами значения, а их элементы.

Типичные поддерживаемые операции:
- проверка соответствия документа поисковому запросу
- проверка вхождения элемента в массив
- поиск документов JSON по ключам или значениям

В столбце "days_of_week" представления "routes" храниться массив номеров дней недели, по которым выполняется рейс:
```sql
SELECT flight_no, days_of_week FROM routes LIMIT 5;

 flight_no | days_of_week 
-----------+--------------
 PG0001    | {6}
 PG0002    | {7}
 PG0003    | {2,6}
 PG0004    | {3,7}
 PG0005    | {2,5,7}
(5 rows)
```

Для представления нельзя построить индекс, поэтому сохраним его строки в отдельной таблице:
```sql
CREATE TABLE routes_tbl AS SELECT * FROM routes;

SELECT 710
```

Теперь создадим GIN-индекс:
```sql
CREATE INDEX routestbl_gin_idx ON routes_tbl USING gin(days_of_week);

CREATE INDEX
```

С помощью GIN-индекса можно, например, отобрать рейсы, отправляющиеся только по средам и субботам:
```sql
EXPLAIN (costs off)
SELECT flight_no, departure_airport_name AS departure, arrival_airport_name AS arrival, days_of_week
FROM routes_tbl WHERE days_of_week = ARRAY[3,6];

                       QUERY PLAN                        
---------------------------------------------------------
 Bitmap Heap Scan on routes_tbl
   Recheck Cond: (days_of_week = '{3,6}'::integer[])
   ->  Bitmap Index Scan on routestbl_gin_idx
         Index Cond: (days_of_week = '{3,6}'::integer[])
(4 rows)
```

Созданный GIN-индекс содержит всего семь элементов: целые числа от 1 до 7, представляющие дни недели.
Для каждого из них в индексе храняться ссылки на рейсы, выполняющиеся в этот день.

Включим обратно последовательное сканирование:
```sql
RESET enable_seqscan;

RESET
```


### Ускорение поиска LIKE

Выполним запрос:
```sql
EXPLAIN (analyze, buffers, costs off, timing off)
SELECT * FROM tickets WHERE contact_data->>'phone' LIKE '%1234%';

                              QUERY PLAN                              
----------------------------------------------------------------------
 Gather (actual rows=1801.00 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   Buffers: shared hit=376 read=49064
   ->  Parallel Seq Scan on tickets (actual rows=600.33 loops=3)
         Disabled: true
         Filter: ((contact_data ->> 'phone'::text) ~~ '%1234%'::text)
         Rows Removed by Filter: 982685
         Buffers: shared hit=376 read=49064
 Planning:
   Buffers: shared hit=6 read=2 dirtied=2
 Planning Time: 3.075 ms
 Execution Time: 622.838 ms
(13 rows)
```

Обратим внимание на количество прочитанных страниц `Buffers` - их почти 50 000.

Добавим расширение `pg_trgm`, которое работает с триграммами.
Представим что есть текстовая строка.
Ее можно разложить на триграммы (последовательность из трех символов) - первый, второй, третий символы; второй, третий, четвертый символы и т.д.
Строка распадается на набор триграмм и эти триграммы хранятся в GIN-индексе.
Такой индекс позволяет ускорять операции поиска вхождения в строку.

```sql
CREATE EXTENSION pg_trgm;

CREATE EXTENSION
```

Создадим GIN-индекс с классом оператов `gin_trgm_ops`:
```sql
CREATE INDEX tickets_gin ON tickets USING gin((contact_data->>'phone') gin_trgm_ops);

CREATE INDEX
```

Такой класс операторов ускоряет поиск по шаблону, который начинается на знак процента, и даже по регулярным выражениям.
Индекс на основе B-дерева этого не позволяет.

Повторим запрос:
```sql
EXPLAIN (analyze, buffers, costs off, timing off)
SELECT * FROM tickets WHERE contact_data->>'phone' LIKE '%1234%';

                                QUERY PLAN                                
--------------------------------------------------------------------------
 Bitmap Heap Scan on tickets (actual rows=1801.00 loops=1)
   Recheck Cond: ((contact_data ->> 'phone'::text) ~~ '%1234%'::text)
   Rows Removed by Index Recheck: 46
   Heap Blocks: exact=1820
   Buffers: shared hit=56 read=1785
   ->  Bitmap Index Scan on tickets_gin (actual rows=1847.00 loops=1)
         Index Cond: ((contact_data ->> 'phone'::text) ~~ '%1234%'::text)
         Index Searches: 1
         Buffers: shared hit=21
 Planning:
   Buffers: shared hit=14 read=2 dirtied=1
 Planning Time: 5.083 ms
 Execution Time: 159.629 ms
(13 rows)
```

Количество прочитанных страниц сократилось более чем на порядок, время выполнения существенно уменьшилось.


## BRIN индекс

BRIN - block range index - индекс зон блоков.

Таблица разбивается на зоны определенной длины, каждая из которых охватывает группу последовательно расположенных страниц.
Размер можно настроить, например 128 табличных страниц в одной зоне.
По каждой зоне собирается сводная информация, минимум и максимум значений индексируемого столбца.

При выполнении запроса пропускаются те зоны, значения в которых не попадают под условие поиска.
А в зонах, которые соответствуют условию, просматриваются все версии строк.
В некотором смысле BRIN - ускоритель последовательного сканирования.


Для примера построим индекс BRIN по самой большой таблице:
```sql
CREATE INDEX tflights_brin_idx ON ticket_flights USING brin(flight_id);

CREATE INDEX
```

Индексы BRIN дают ощутимый эффект только для очень больших таблиц.
Тем не менее, планировщик использует построенный индекс, поскольку по сводной информации (минимум и максимум) можно отсечь зоны, не содержащие требуемых значений:
```sql
EXPLAIN (analyze, costs off, timing off)
SELECT * FROM ticket_flights WHERE flight_id BETWEEN 3000 AND 4000;

                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Gather (actual rows=46357.00 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   Buffers: shared hit=49 read=59848
   ->  Parallel Bitmap Heap Scan on ticket_flights (actual rows=15452.33 loops=3)
         Recheck Cond: ((flight_id >= 3000) AND (flight_id <= 4000))
         Rows Removed by Index Recheck: 2377352
         Heap Blocks: lossy=21341
         Buffers: shared hit=49 read=59848
         Worker 0:  Heap Blocks: lossy=19332
         Worker 1:  Heap Blocks: lossy=19175
         ->  Bitmap Index Scan on tflights_brin_idx (actual rows=598480.00 loops=1)
               Index Cond: ((flight_id >= 3000) AND (flight_id <= 4000))
               Index Searches: 1
               Buffers: shared hit=9
 Planning:
   Buffers: shared hit=27 read=6
 Planning Time: 3.878 ms
 Execution Time: 1139.608 ms
(19 rows)
```

Поскольку BRIN не хранит ссылки на версии строк, единственный возможный способ доступа - сканирование по неточной (`lossy`) битовой карте.
