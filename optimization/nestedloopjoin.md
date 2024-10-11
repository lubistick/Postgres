# Соединения

Способы соединения - не соединения SQL
inner/left/right/full/cross join/in/exists - логические операции
способы соединения - механизм реализации

Соединяются не таблицы, а наборы строк
могут быть получены от любого узла дерева плана

Наборы строк соединяются попарно
порядок соединений важен с точки зрения производительности
обычно важен и порядок внутри пары

SELECT a.title, s.name FROM albums a JOIN songs s ON a.id = s.album_id;
для каждой строки одного набора
перебираем подходящие строки другого набора
индексный доступ или полное сканирование

Вычислительная сложность
N * M
где N и M - число строк во внешнем и внутреннем наборах данных
эффективно только для небольшого числа строк

В параллельном режиме
внешний набор строк сканируется параллельно,
внутренний - последовательно одним из рабочих процессов


## Вложенный цикл (Nested Loop)

Так выглядит план для соединения вложенным циклом,
которое оптимизатор обычно предпочитает для небольших выборок (смотрим перелеты, включенные в два билета):

```sql
EXPLAIN (costs off)
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
- Узел `Nested Loop` обращается к первому (внешнему) набору за первой строкой.
Здесь это - узел `Index Scan` по билетам `tickets`.
- Узел `Nested Loop` обращается ко второму (внутреннему) набору и просит выдать все строки,
соответствующие строке первого набора.
Здесь это - узел `Index Scan` по перелетам `ticket_flights`.
- Процесс повторяется до тех пор, пока внешний набор не исчерпает все строки.


Посмотрим на оцененую стоимость.

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
Такой способ может пригодиться, т.к. FULL JOIN (как мы увидим позже) работает только с эквисоединениями.


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
