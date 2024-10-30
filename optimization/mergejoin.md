# Соединение слиянием

Стоимость соединения слиянием пропорциональна `N + M`, где `N` и `M` - число строк в первом и втором наборах данных, если не требуется сортировка.
Сортировка набора из `K` строк добавляет к оценке как минимум `K * log(K)`.


Создание индекса.

Индексная сортировка.
Сначала все строки сортируются.
Затем строки собираются в листовые индексные страницы.
Ссылки на них собираются в страницы следующего уровня и т.д., пока не дойдем до корня.

Ограничение.
maintenance_work_mem, т.к. операция не частая.
Для уникальных индексов требуется дополнительно work_mem.


## Алгоритм Merge Join

Чтобы начать соединение, необходимо чтобы оба набора данных были отсортированы по ключу соединения.
Сначала берутся первая строка первого набора и первая строка второго набора, и сравниваются между собой.
Если есть соответствие по ключу соединения, то результат соединения возвращается и читается следующая строка второго набора.
Если нет соответствия, то читается следующая строка того набора, для которого значение поля, по которому происходит соединение, меньше.
Т.е. один набор "догоняет другой". И так пока наборы не будут прочитаны.
Алгоритм слияния возвращает результат соединения в отсортированном виде.

Если результат необходим в отсортированном виде (используется `ORDER BY`), оптимизатор может предпочесть соединение слиянием.
Особенно, если данные от дочерних узлов можно получить уже отсортированными с помощью индексного сканирования:
```sql
EXPLAIN (COSTS OFF) SELECT * FROM tickets t
JOIN ticket_flights tf ON tf.ticket_no = t.ticket_no ORDER BY t.ticket_no;

                           QUERY PLAN                            
-----------------------------------------------------------------
 Merge Join
   Merge Cond: (t.ticket_no = tf.ticket_no)
   ->  Index Scan using tickets_pkey on tickets t
   ->  Index Scan using ticket_flights_pkey on ticket_flights tf
(4 rows)
```


### Стоимость

Посмотрим на стоимость:
```sql
EXPLAIN SELECT * FROM tickets t
JOIN ticket_flights tf ON tf.ticket_no = t.ticket_no ORDER BY t.ticket_no;

                                                  QUERY PLAN                                                   
---------------------------------------------------------------------------------------------------------------
 Merge Join  (cost=8.44..822282.10 rows=8391960 width=136)
   Merge Cond: (t.ticket_no = tf.ticket_no)
   ->  Index Scan using tickets_pkey on tickets t  (cost=0.43..139110.29 rows=2949857 width=104)
   ->  Index Scan using ticket_flights_pkey on ticket_flights tf  (cost=0.56..570902.89 rows=8391960 width=32)
 JIT:
   Functions: 7
   Options: Inlining true, Optimization true, Expressions true, Deforming true
(7 rows)
```

Начальная стоимость узла `Merge Join` включает:
- сумму начальных стоимостей дочерних узлов (в примере это 2 узла `Index Scan`), также стоимость сортировки, если она необходима.
- стоимость получения первой пары строк, соответствующих друг другу.

Полная стоимость складывается из:
- суммы стоимостей получения обоих наборов данных.
- стоимости сравнения строк.


В отличие от соединения хешированием, слияние без сортировки хорошо подходит для случая, когда надо быстро получить первые строки:
```sql
EXPLAIN SELECT * FROM tickets t
JOIN ticket_flights tf ON tf.ticket_no = t.ticket_no ORDER BY t.ticket_no LIMIT 1000;

                                                     QUERY PLAN                                                      
---------------------------------------------------------------------------------------------------------------------
 Limit  (cost=8.44..106.42 rows=1000 width=136)
   ->  Merge Join  (cost=8.44..822282.10 rows=8391960 width=136)
         Merge Cond: (t.ticket_no = tf.ticket_no)
         ->  Index Scan using tickets_pkey on tickets t  (cost=0.43..139110.29 rows=2949857 width=104)
         ->  Index Scan using ticket_flights_pkey on ticket_flights tf  (cost=0.56..570902.89 rows=8391960 width=32)
(5 rows)
```

Общая стоимость значительно уменьшилась, т.к. не надо перебирать наборы до конца.

Если нужного индекса не окажется, Postgres выполнит сортировку в узле `Sort`.
Тогда начальная стоимость узла `Merge Join` будет не меньше общей стоимости сортировки, и первые строки запрос не сможет выдавать без задержки.
Пример такого плана:
```sql
EXPLAIN SELECT * FROM aircrafts a JOIN seats s ON a.aircraft_code = s.aircraft_code ORDER BY a.aircraft_code;

                                     QUERY PLAN                                      
-------------------------------------------------------------------------------------
 Merge Join  (cost=1.51..420.71 rows=1339 width=67)
   Merge Cond: (s.aircraft_code = ml.aircraft_code)
   ->  Index Scan using seats_pkey on seats s  (cost=0.28..64.60 rows=1339 width=15)
   ->  Sort  (cost=1.23..1.26 rows=9 width=52)
         Sort Key: ml.aircraft_code
         ->  Seq Scan on aircrafts_data ml  (cost=0.00..1.09 rows=9 width=52)
(6 rows)
```


## Группировка и уникальные значения

Как мы видели, для устранения дубликатов может использоваться  хеширование.
Другой способ - сортировка значений:
```sql
EXPLAIN SELECT DISTINCT book_date FROM bookings ORDER BY book_date;
```

Сортировка и устранение дубликатов происходит в узле Unique.
Для группировки будет использоваться узел с другим названием - GroupAggregate.

Такой способ особенно выгоден, если требуется получить отсортированный результат (как в данном случае).


## Использование памяти при сортировке

Еще один случай, когда сортировка оказывается выгодней - ограниченная оперативная память.
Алгоритм внешней сортировки работает эффективней, чем хеш-соединение и использованием нескольких пакетов.

Включим сообщения о временных файлах и посмотрим, как выполняется сортировка.
```sql
SET log_temp_files = 0;
```

```sql
EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE) SELECT DISTINCT book_date FROM bookings;
```

Строки не помещаются в доступную память и используется внешняя сортировка (Sort Method: external merge).

Также в журнале сообщений мы видим запись о временном файле:
```bash
tail -n 4 /val/log/....исправить путь
```

Запись появляется, когда файл освобождается, поэтому его размер известен.

Теперь увеличим work_mem:
```sql
SET work_mem = '32MB';
```

Выполним тот же запрос:
```sql
EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE) SELECT DISTINCT book_date FROM bookings;
```

Поскольку сортировка не требуется и памяти достаточно, планировщик переключился на использование хеширования.

Но если добавить предложение ORDER BY, планировщик возвращается к сортировке.
Он будет строить хеш-таблицу, а ее результат в конце отсортирует:
```sql
EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE) SELECT DISTINCT book_date FROM bookings ORDER BY book_date;
```

Теперь все строки поместились в память (Sort Method: quicksort).


## Соединение нескольких таблиц

Пример запроса с двумя соединениями слиянием (номера билетов и места в салоне):
```sql
EXPLAIN (COSTS OFF) SELECT t.ticket_no, bp.flight_id, bp.seat_no FROM tickets t
JOIN ticket_flights tf ON t.ticket_no = tf.ticket_no
JOIN boarding_passes bp ON bp.ticket_no = tf.ticket_no AND bp.flight_id = tf.flight_id
ORDER BY t.ticket_no;
```

Здесь соединяются tickets (билеты) и boarding_passes (посадочные талоны),
и с этим, уже отсортированным по номерам билетов, набором строк соединяются ticket_flights (перелеты).


## Параллельные планы

(дописать)
