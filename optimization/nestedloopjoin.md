# Соединение вложеннымы циклом

Алгоритм `Nested Loop`: для каждой строки одного из наборов перебираем и возвращаем соответствующие ему строки второго набора.
По сути - цикл в цикле. Оптимизатор предпочитает `Nested Loop` для небольших выборок.


## Классический алгоритм

Посмотрим перелеты, включенные в два конкретных билета:

```sql
EXPLAIN (COSTS OFF)
SELECT * FROM tickets t
JOIN ticket_flights tf ON tf.ticket_no = t.ticket_no
WHERE t.ticket_no IN ('0005432312163', '0005432312164');

                                    QUERY PLAN                                     
-----------------------------------------------------------------------------------
 Nested Loop
   ->  Index Scan using tickets_pkey on tickets t
         Index Cond: (ticket_no = ANY ('{0005432312163,0005432312164}'::bpchar[]))
   ->  Index Scan using ticket_flights_pkey on ticket_flights tf
         Index Cond: (ticket_no = t.ticket_no)
(5 rows)
```

План запроса:
- Узел `Nested Loop` обращается за данными к внешний узлу `Index Scan` по таблице `tickets` (билеты).
- Внешний `Index Scan` отдает ОДНУ строку узлу `Nested Loop`, подходящую под условие `Index Cond`.
- Узел `Nested Loop` обращается за данными по конкретной строке к внутреннему узлу `Index Scan` по таблице `ticket_flights` (перелеты).
- Внутренний `Index Scan` ищет ВСЕ СТРОКИ, подходящую под условие `Index Cond`, и отдает их узлу `Nested Loop`.
- Процесс повторяется до тех пор, пока не закончатся строки для внешнего узла.


### Оценка стоимости

Посмотрим на оценку стоимости:

```sql
EXPLAIN
SELECT * FROM tickets t
JOIN ticket_flights tf ON tf.ticket_no = t.ticket_no
WHERE t.ticket_no IN ('0005432312163', '0005432312164');

                                             QUERY PLAN                                              
-----------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.99..46.10 rows=6 width=136)
   ->  Index Scan using tickets_pkey on tickets t  (cost=0.43..12.89 rows=2 width=104)
         Index Cond: (ticket_no = ANY ('{0005432312163,0005432312164}'::bpchar[]))
   ->  Index Scan using ticket_flights_pkey on ticket_flights tf  (cost=0.56..16.57 rows=3 width=32)
         Index Cond: (ticket_no = t.ticket_no)
(5 rows)
```

- Начальная стоимость `Nested Loop` (`0.99`) - сумма первых компонент стоимости дочерних узлов `Index Scan` (`0.43` и `0.56`).
- Конечная стоимость `Nested Loop` складывается из:
    - стоимости получения данных от внешнего набора (вторая компонента `Index Scan` по билетам `tickets`)
    - стоимости получения данных от внутреннего набора (вторая компонента `Index Scan` по перелетам `ticket_flights`), умноженной на число строк внешнего набора (`2` строки)
    - процессорного времени на обработку строк

В общем случае формула более сложная, но основной вывод: стоимость пропорциональна `N * M`, где `N` и `M` - число строк в соединяемых наборах данных.

Для непараметризованного соединения `M` будет в точности равно числу строк внутреннего набора; для параметризованного — `M` может быть значительно меньше.


Выполним `EXPLAIN ANALYZE`:

```sql
EXPLAIN (costs off, ANALYZE)
SELECT * FROM tickets t
JOIN ticket_flights tf ON tf.ticket_no = t.ticket_no
WHERE t.ticket_no IN ('0005432312163', '0005432312164');

                                                QUERY PLAN                                                 
-----------------------------------------------------------------------------------------------------------
 Nested Loop (actual time=3.602..4.898 rows=8 loops=1)
   ->  Index Scan using tickets_pkey on tickets t (actual time=0.066..0.085 rows=2 loops=1)
         Index Cond: (ticket_no = ANY ('{0005432312163,0005432312164}'::bpchar[]))
   ->  Index Scan using ticket_flights_pkey on ticket_flights tf (actual time=1.770..2.396 rows=4 loops=2)
         Index Cond: (ticket_no = t.ticket_no)
 Planning Time: 0.572 ms
 Execution Time: 4.974 ms
(7 rows)
```

Плане запроса:
- `loops` имеет значение `1` - сколько раз выполнялся вложенный цикл.
- `rows` имеет значение `8` - сколько в среднем было выбрано строк. Видно, что планировщик немного ошибся - получилось `8` строк вместо `6`.
- `time` - сколько потрачено времени за один раз.

Предупреждение: вывод времени выполнения каждого шага, как в этом примере, может существенно замедлять выполнение запроса на некоторых платформах.
Если такая информация не нужна, лучше указывать фразу `TIMING OFF`.

Выведем самолеты, способные обслуживать перелеты заданной протяженности:

```sql
EXPLAIN (costs off)
SELECT * FROM (VALUES (1000), (10000)) d(range)
JOIN aircrafts a ON a.range >= d.range;

                   QUERY PLAN
-------------------------------------------------
 Nested Loop
   Join Filter: (ml.range >= "*VALUES*".column1)
   ->  Seq Scan on aircrafts_data ml
   ->  Materialize
         ->  Values Scan on "*VALUES*"
(5 rows)
```

Соединение вложенным циклом позволяет соединять строки по любому условию, не обязательно по равенству значений.

## Модификации

Существует несколько модификаций алгоритма `Nested Loop`.

### Nested Loop Left Join
 
Модификация `Left Join` возвращает строки левого набора, даже если им не нашлось соответствия в правом наборе:

```sql
EXPLAIN (COSTS OFF) SELECT * FROM aircrafts a
LEFT JOIN seats s ON a.aircraft_code = s.aircraft_code
WHERE a.model LIKE 'Аэробус%';

                          QUERY PLAN                          
--------------------------------------------------------------
 Nested Loop Left Join
   ->  Seq Scan on aircrafts_data ml
         Filter: ((model ->> lang()) ~~ 'Аэробус%'::text)
   ->  Bitmap Heap Scan on seats s
         Recheck Cond: (ml.aircraft_code = aircraft_code)
         ->  Bitmap Index Scan on seats_pkey
               Index Cond: (aircraft_code = ml.aircraft_code)
(7 rows)
```

Такая же модификация есть и для правого соединения `Right Join`.
Планировщик сам определяет порядок соединения таблиц, независимо от их порядка в запросе.


### Nested Loop Anti Join

Модификация `Anti Join` возвращает строки одного набора, если для них нет соответствия в другом наборе:

```sql
EXPLAIN (COSTS OFF) SELECT * FROM aircrafts a
WHERE a.model LIKE 'Аэробус%' AND NOT EXISTS (
      SELECT * FROM seats s WHERE s.aircraft_code = a.aircraft_code
);

                        QUERY PLAN                        
----------------------------------------------------------
 Nested Loop Anti Join
   ->  Seq Scan on aircrafts_data ml
         Filter: ((model ->> lang()) ~~ 'Аэробус%'::text)
   ->  Index Only Scan using seats_pkey on seats s
         Index Cond: (aircraft_code = ml.aircraft_code)
(5 rows)
```

Модификация `Anti Join` используется и для аналогичного запроса, записанного иначе:

```sql
EXPLAIN (COSTS OFF) SELECT * FROM aircrafts a
LEFT JOIN seats s ON a.aircraft_code = s.aircraft_code
WHERE a.model LIKE 'Аэробус%' AND s.aircraft_code IS NULL;

                        QUERY PLAN                        
----------------------------------------------------------
 Nested Loop Anti Join
   ->  Seq Scan on aircrafts_data ml
         Filter: ((model ->> lang()) ~~ 'Аэробус%'::text)
   ->  Index Scan using seats_pkey on seats s
         Index Cond: (aircraft_code = ml.aircraft_code)
(5 rows)
```


### Nested Loop Semi Join

Модификация `Semi Join` возвращает строки одного набора, если для них нашлось хотя бы одно соответствие в другом наборе:

```sql
EXPLAIN SELECT * FROM aircrafts a
WHERE a.model LIKE 'Аэробус%' AND EXISTS (
      SELECT * FROM seats s WHERE s.aircraft_code = a.aircraft_code
);

                                      QUERY PLAN                                       
---------------------------------------------------------------------------------------
 Nested Loop Semi Join  (cost=0.28..4.02 rows=1 width=52)
   ->  Seq Scan on aircrafts_data ml  (cost=0.00..3.39 rows=1 width=52)
         Filter: ((model ->> lang()) ~~ 'Аэробус%'::text)
   ->  Index Only Scan using seats_pkey on seats s  (cost=0.28..6.88 rows=149 width=4)
         Index Cond: (aircraft_code = ml.aircraft_code)
(5 rows)
```

Обратим внимание, в плане для таблицы `seats` указано `rows` равным `149`.
На самом деле, достаточно получить всего одну строку, чтобы понять значение предиката `EXISTS`.

Postgres так и делает:

```sql
EXPLAIN (COSTS OFF, ANALYZE) SELECT * FROM aircrafts a
WHERE a.model LIKE 'Аэробус%' AND EXISTS (
      SELECT * FROM seats s WHERE s.aircraft_code = a.aircraft_code
);

                                         QUERY PLAN                                          
---------------------------------------------------------------------------------------------
 Nested Loop Semi Join (actual time=9.200..9.256 rows=3 loops=1)
   ->  Seq Scan on aircrafts_data ml (actual time=6.407..6.438 rows=3 loops=1)
         Filter: ((model ->> lang()) ~~ 'Аэробус%'::text)
         Rows Removed by Filter: 6
   ->  Index Only Scan using seats_pkey on seats s (actual time=0.928..0.928 rows=1 loops=3)
         Index Cond: (aircraft_code = ml.aircraft_code)
         Heap Fetches: 0
 Planning Time: 0.199 ms
 Execution Time: 9.316 ms
(9 rows)
```

### Модификации для FULL JOIN не существует

Это связано с тем, что полный проход по второму набору строк может не выполняться.
Если пренебречь производительностью, полное соединение можно получить, объединив левое соединение и антисоединение.
Такой способ может пригодиться, т.к. FULL JOIN (как мы увидим позже) работает только с эквисоединениями (по условию равенства).


## Мемоизация

Если внутренний набор сканируется много раз с повторяющимися значениями параметров, иногда имеет смысл закешировать результат.
Эта операция называется мемоизацией.
Для кеширования используется хеш-таблица.

Посмотрим план запроса:

```sql
EXPLAIN (analyze, buffers off, costs off, timing off, summary off)
SELECT * FROM flights f
JOIN aircrafts_data a ON f.aircraft_code = a.aircraft_code
WHERE f.flight_no = 'PG0003';

                                               QUERY PLAN
---------------------------------------------------------------------------------------------------------
 Nested Loop (actual rows=113.00 loops=1)
   ->  Bitmap Heap Scan on flights f (actual rows=113.00 loops=1)
         Recheck Cond: (flight_no = 'PG0003'::bpchar)
         Heap Blocks: exact=2
         ->  Bitmap Index Scan on flights_flight_no_scheduled_departure_key (actual rows=113.00 loops=1)
               Index Cond: (flight_no = 'PG0003'::bpchar)
               Index Searches: 1
   ->  Memoize (actual rows=1.00 loops=113)
         Cache Key: f.aircraft_code
         Cache Mode: logical
         Hits: 112  Misses: 1  Evictions: 0  Overflows: 0  Memory Usage: 1kB
         ->  Index Scan using aircrafts_pkey on aircrafts_data a (actual rows=1.00 loops=1)
               Index Cond: (aircraft_code = f.aircraft_code)
               Index Searches: 1
(14 rows)
```

Пример подобран таким образом, что из внутреннего набора (таблица `aircrafts_data`) будет получена только одна строка по ключу `f.aircraft_code` (`Cache Key`), она и будет закеширована в узле `Memoize`:

- `Misses` - сколько раз приходилось обращаться в таблицу - `1`
- `Hits` - сколко раз значение нашлось в кеше - `112`
- `Evictions` имеет значение `0` - вытеснений из кеша не было
- `Overflows` имеет значение `0` - строки, выбранные из внутреннего набора, всегда умещались в выделенную память

Если закешированные строки не помещаются в память, которая ограничена размером `work_mem` умножить на `hash_mem_multiplier`,
значение параметра игнорируется, поскольку кешировать только часть строк нет смысла.
В плане запроса количество таких ситуаций будет показано как `Overflows`.


## Соединение нескольких таблиц.

Найдем всех пассажиров, купивших билеты на определенный рейс:

```sql
EXPLAIN (COSTS OFF) SELECT t.passenger_name FROM tickets t
JOIN ticket_flights tf ON tf.ticket_no = t.ticket_no
JOIN flights f ON f.flight_id = tf.flight_id
WHERE f.flight_id = 12345;

                          QUERY PLAN                          
--------------------------------------------------------------
 Nested Loop
   ->  Index Only Scan using flights_pkey on flights f
         Index Cond: (flight_id = 12345)
   ->  Gather
         Workers Planned: 2
         ->  Nested Loop
               ->  Parallel Seq Scan on ticket_flights tf
                     Filter: (flight_id = 12345)
               ->  Index Scan using tickets_pkey on tickets t
                     Index Cond: (ticket_no = tf.ticket_no)
(10 rows)
```

На верхнем уровне используется соединение вложенным циклом.
Внешний набор данных состоит из одной строки, полученной из таблицы `flights` (рейсы) по уникальному индексу.
Для получения внутреннего набора используется параллельный план.
Каждый из процессов читает свою часть таблицы `tickets_flights` (перелеты)
и соединяет ее с таблицей `tickets` (билеты) с помощью другого вложенного цикла.


## Практика

Создадим индекс на таблице рейсов:

```sql
CREATE INDEX ON flights(departure_airport);

CREATE INDEX
```

Найдем все рейсы из Ульяновска:
```sql
EXPLAIN SELECT * FROM flights f
JOIN airports a ON a.airport_code = f.departure_airport
WHERE a.city = 'Ульяновск';

                                              QUERY PLAN                                              
------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=24.31..3732.03 rows=2066 width=162)
   ->  Seq Scan on airports_data ml  (cost=0.00..30.56 rows=1 width=145)
         Filter: ((city ->> lang()) = 'Ульяновск'::text)
   ->  Bitmap Heap Scan on flights f  (cost=24.31..2637.48 rows=2066 width=63)
         Recheck Cond: (departure_airport = ml.airport_code)
         ->  Bitmap Index Scan on flights_departure_airport_idx  (cost=0.00..23.79 rows=2066 width=0)
               Index Cond: (departure_airport = ml.airport_code)
(7 rows)
```

Планировщик использовал соединение вложенным циклом.
Заметим, что в данном случае соединение необходимо, т.к. в Ульяновске два аэропорта:

```sql
SELECT * FROM airports WHERE city = 'Ульяновск';

 airport_code |    airport_name     |   city    |              coordinates               |   timezone    
--------------+---------------------+-----------+----------------------------------------+---------------
 ULY          | Ульяновск-Восточный | Ульяновск | (48.80270004272461,54.4010009765625)   | Europe/Samara
 ULV          | Баратаевка          | Ульяновск | (48.226699829100006,54.26829910279999) | Europe/Samara
(2 rows)
```


### Декартово произведение

Соединим две таблицы без указания условий соединения, иными словами, выполним декартово произведение таблиц:

```sql
EXPLAIN SELECT * FROM airports a1 CROSS JOIN airports a2;

                                    QUERY PLAN                                    
----------------------------------------------------------------------------------
 Nested Loop  (cost=0.00..11067.70 rows=10816 width=198)
   ->  Seq Scan on airports_data ml  (cost=0.00..4.04 rows=104 width=145)
   ->  Materialize  (cost=0.00..4.56 rows=104 width=145)
         ->  Seq Scan on airports_data ml_1  (cost=0.00..4.04 rows=104 width=145)
(4 rows)
```

Соединение вложенным циклом - единственный способ выполнение таких соединений.

Узел `Materialize` - "материализация" выборки строк.
Если выборка не превышает размер, заданный параметром `work_mem`, то она остается в памяти.
Если превышает - записывается во временный файл.
В данном случае эта операция выглядит лишней, но в следующем примере она позволяет сканировать во вложенном цикле уже отфильтрованный набор строк из `a2`, а не всю таблицу:

```sql
EXPLAIN SELECT * FROM airports a1 CROSS JOIN airports a2 WHERE a2.timezone = 'Europe/Moscow';

                                   QUERY PLAN                                    
---------------------------------------------------------------------------------
 Nested Loop  (cost=0.00..4687.41 rows=4576 width=198)
   ->  Seq Scan on airports_data ml  (cost=0.00..4.04 rows=104 width=145)
   ->  Materialize  (cost=0.00..4.52 rows=44 width=145)
         ->  Seq Scan on airports_data ml_1  (cost=0.00..4.30 rows=44 width=145)
               Filter: (timezone = 'Europe/Moscow'::text)
(5 rows)
```


### Расстояния между аэропортами

Построим таблицу расстояний между всеми аэропортами так, чтобы каждая пара встречалась только один раз.
Нам понадобится расширение `earthdistance`:

```sql
CREATE EXTENSION earthdistance CASCADE;

NOTICE:  installing required extension "cube"
CREATE EXTENSION
```

Чтобы отсечь повторяющиеся пары, соединим таблицы по условию `>` (больше):

```sql
EXPLAIN
SELECT a1.airport_code "from", a2.airport_code "to", a1.coordinates <@> a2.coordinates "distance, miles"
FROM airports a1 JOIN airports a2 ON a1.airport_code > a2.airport_code;
                                             QUERY PLAN
-----------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.14..147.97 rows=3605 width=16)
   ->  Seq Scan on airports_data ml  (cost=0.00..4.04 rows=104 width=20)
   ->  Index Scan using airports_data_pkey on airports_data ml_1  (cost=0.14..0.95 rows=35 width=20)
         Index Cond: (airport_code < ml.airport_code)
(4 rows)
```

Вложенный цикл — единственный способ соединения для такого условия.
