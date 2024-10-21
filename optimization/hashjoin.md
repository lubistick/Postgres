# Соединение хешированием

Вычислительная сложность N + M, где N и M - число строк в первом и втором наборах данных.
Начальные затраты на построение хеш-таблицы.
Эффективно для большого числа строк.

Оперативная память.
Память каждой операции ограничена параметром work_mem.
При выполнении запроса несколько операций могут использовать память одновременно.
Если выделенной памяти не хватает, используется диск (иногда операция может и превысить ограничение).
Больше памяти - быстрее выполнение.

Дисковая память.
Общая дисковая память сеанса ограничена параметром temp_file_limit (без учета временных таблиц).


Размер хеш-таблицы в памяти ограничен значением work_mem * hash_mem_multiplier.


## Алгоритм Hash Join

Для большой выборки оптимизатор предпочитает соединение хешированием.
```sql
EXPLAIN (COSTS OFF) SELECT * FROM tickets t
JOIN ticket_flights tf ON tf.ticket_no = t.ticket_no;

                QUERY PLAN                 
-------------------------------------------
 Hash Join
   Hash Cond: (tf.ticket_no = t.ticket_no)
   ->  Seq Scan on ticket_flights tf
   ->  Hash
         ->  Seq Scan on tickets t
(5 rows)
```

Сначала выполняется узел `Seq Scan` по таблице `tickets` (билеты).
Для каждой строки `tickets` вычисляется хеш-функция от значений полей, входящих в условие соединения (поле `ticket_no`).
В корзину хеш-таблицы помещаются вычисленный хеш-код и все поля,
которые входят в условие соединения (поле `ticket_no`) или используются в запросе (все поля двух таблиц, т.к. указана `*`).

Затем выполняется `Seq Scan` по таблице `ticket_flights` (перелеты).
Для каждой строки `ticket_flights` вычисляется хеш-функция.
Проверяется корзина, соответствующая вычисленному значению.
Если есть корзина, то проверяются значения полей в условиях соединения.

Проверки только хеш-кода не достаточно, т.к. возможны коллизии, при которых разные значения получат одинаковые хеш-коды.


### Стоимость

Посмотрим на стоимость:
```sql
EXPLAIN SELECT * FROM tickets t JOIN ticket_flights tf ON tf.ticket_no = t.ticket_no;

                                    QUERY PLAN                                     
-----------------------------------------------------------------------------------
 Hash Join  (cost=161878.78..498586.31 rows=8391960 width=136)
   Hash Cond: (tf.ticket_no = t.ticket_no)
   ->  Seq Scan on ticket_flights tf  (cost=0.00..153852.60 rows=8391960 width=32)
   ->  Hash  (cost=78913.57..78913.57 rows=2949857 width=104)
         ->  Seq Scan on tickets t  (cost=0.00..78913.57 rows=2949857 width=104)
 JIT:
   Functions: 10
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(8 rows)
```

Начальная стоимость узла `Hash Join` складывается из:
- стоимости получения всего первого набора данных, а именно `Seq Scan` по таблице `tickets`.
- стоимости построения хеш-таблицы - пока таблица не готова, соединение не может начаться.
В узле `Hash` не отражена стоимость построения хеш-таблицы.

Полная стоимость складывается из:
- стоимости получения всего второго набора данных, а именно `Seq Scan` по таблице `tickets`.
- стоимости проверки по хеш-таблице.
- стоимости обращения к диску в случае, когда предполагается использование более одного пакета.

Стоимость хеш-соединения пропорциональна `N + M`, где `N` и `M` - число строк в соединяемых наборах данных.
При больших `N` и `M` соединение хешированием (`Hash Join`) значительно выгоднее, чем произведение в случае соединения вложенным циклом (`Nested Loop`).


## Модификации

Модификации `Hash Join` включают `Left`, `Semi`, `Anti`, а также `Full` для полного соединения:
```sql
EXPLAIN SELECT * FROM aircrafts a FULL JOIN seats s ON a.aircraft_code = s.aircraft_code;

                                  QUERY PLAN                                  
------------------------------------------------------------------------------
 Hash Full Join  (cost=3.47..30.04 rows=1339 width=67)
   Hash Cond: (s.aircraft_code = ml.aircraft_code)
   ->  Seq Scan on seats s  (cost=0.00..21.39 rows=1339 width=15)
   ->  Hash  (cost=3.36..3.36 rows=9 width=52)
         ->  Seq Scan on aircrafts_data ml  (cost=0.00..3.36 rows=9 width=52)
(5 rows)
```


## Группировка и уникальные значения

Для группировки (`GROUP BY`) и устранения дубликатов (`DISTINCT` и операции со множествами без слова `ALL`) используются методы, схожие с методами соединения.
Один из способов выполнения состоит в том, чтобы построить хеш-таблицу по нужным полям и получить из нее уникальные значения:

```sql
EXPLAIN SELECT fare_conditions, count(*) FROM seats GROUP BY fare_conditions;
                          QUERY PLAN                           
---------------------------------------------------------------
 HashAggregate  (cost=28.09..28.12 rows=3 width=16)
   Group Key: fare_conditions
   ->  Seq Scan on seats  (cost=0.00..21.39 rows=1339 width=8)
(3 rows)
```

Тоже самое и с `DISTINCT`:
```sql
EXPLAIN SELECT DISTINCT fare_conditions FROM seats;

                          QUERY PLAN                           
---------------------------------------------------------------
 HashAggregate  (cost=24.74..24.77 rows=3 width=8)
   Group Key: fare_conditions
   ->  Seq Scan on seats  (cost=0.00..21.39 rows=1339 width=8)
(3 rows)
```


## Использование памяти

Посмотрим значение `work_mem`:
```sql
SHOW work_mem;

 work_mem 
----------
 4MB
(1 row)
```

Посмотрим значение `hash_mem_multiplier`:
```sql
SHOW hash_mem_multiplier;

 hash_mem_multiplier 
---------------------
 1
(1 row)
```

Размер памяти, отведенной под хеш-таблицу:
```sql
SELECT
    wm.setting work_mem,
    wm.unit,
    hmm.setting hash_mem_multiplier,
    wm.setting::numeric * hmm.setting::numeric total
FROM pg_settings wm, pg_settings hmm
WHERE
    wm.name = 'work_mem' AND
    hmm.name = 'hash_mem_multiplier';

 work_mem | unit | hash_mem_multiplier | total 
----------+------+---------------------+-------
 4096     | kB   | 1                   |  4096
(1 row)
```

Увеличим размер памяти, отведенной под хеш-таблицу:
```sql
SET work_mem = '64MB';

SET
```

```sql
SET hash_mem_multiplier = 3;

SET
```

```sql
SELECT
    wm.setting work_mem,
    wm.unit,
    hmm.setting hash_mem_multiplier,
    wm.setting::numeric * hmm.setting::numeric total
FROM pg_settings wm, pg_settings hmm
WHERE
    wm.name = 'work_mem' AND
    hmm.name = 'hash_mem_multiplier';

 work_mem | unit | hash_mem_multiplier | total  
----------+------+---------------------+--------
 65536    | kB   | 3                   | 196608
(1 row)
```

Команда `EXPLAIN` показывает нестандартные значения параметров при указании `SETTINGS`:
```sql
EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE, SETTINGS, SUMMARY OFF) SELECT * FROM bookings b
JOIN tickets t ON b.book_ref = t.book_ref;

                                        QUERY PLAN                                        
------------------------------------------------------------------------------------------
 Hash Join (actual rows=2949857 loops=1)
   Hash Cond: (t.book_ref = b.book_ref)
   ->  Seq Scan on tickets t (actual rows=2949857 loops=1)
   ->  Hash (actual rows=2111110 loops=1)
         Buckets: 4194304  Batches: 1  Memory Usage: 145986kB
         ->  Seq Scan on bookings b (actual rows=2111110 loops=1)
 Settings: hash_mem_multiplier = '3', search_path = 'bookings, public', work_mem = '64MB'
(7 rows)
```
Улез `Hash`:
- `Buckets` - число корзин в хеш-таблице
- `Batches` - имеет значение `1` - вся хеш-таблица поместилась в память
- `Memory Usage` - `145 986 kB` - использованная оперативная память

Хеш-таблица строилась по меньшему набору строк.

Сравним с таким же запросом, который выводит только одно поле:
```sql
EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE, SETTINGS, SUMMARY OFF) SELECT b.book_ref FROM bookings b
JOIN tickets t ON b.book_ref = t.book_ref;

                                        QUERY PLAN                                        
------------------------------------------------------------------------------------------
 Hash Join (actual rows=2949857 loops=1)
   Hash Cond: (t.book_ref = b.book_ref)
   ->  Seq Scan on tickets t (actual rows=2949857 loops=1)
   ->  Hash (actual rows=2111110 loops=1)
         Buckets: 4194304  Batches: 1  Memory Usage: 113172kB
         ->  Seq Scan on bookings b (actual rows=2111110 loops=1)
 Settings: hash_mem_multiplier = '3', search_path = 'bookings, public', work_mem = '64MB'
(7 rows)
```

Расход памяти уменьшился `113 172 kB`, т.к. в хем-таблице теперь только одно поле, вместо трех.

- `Hash Cond` - содержит предикаты, участвующие в соединении.

Добавим дополнительное условие соединения:
```sql
EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE, SUMMARY OFF) SELECT b.book_ref FROM bookings b
JOIN tickets t ON b.book_ref = t.book_ref AND b.total_amount::text > t.passenger_id;

                            QUERY PLAN                            
------------------------------------------------------------------
 Hash Join (actual rows=1198623 loops=1)
   Hash Cond: (t.book_ref = b.book_ref)
   Join Filter: ((b.total_amount)::text > (t.passenger_id)::text)
   Rows Removed by Join Filter: 1751234
   ->  Seq Scan on tickets t (actual rows=2949857 loops=1)
   ->  Hash (actual rows=2111110 loops=1)
         Buckets: 4194304  Batches: 1  Memory Usage: 127431kB
         ->  Seq Scan on bookings b (actual rows=2111110 loops=1)
(8 rows)
```

Дополнительные условия соединения `Join Filter` также попадают в хеш-таблицу.
`Memory Usage` вырос до `127 431 kB`.


### Двухпроходное соединение

Применяется, когда хеш-таблица не помещается в оперативную память.
Наборы данных разбиваются на пакеты и последовательно соединяются.

Уменьшим `work_mem` и `hash_mem_multiplier` так, чтобы хеш-таблица не поместилась в оперативную память.
```sql
SET work_mem = '32MB';

SET
```

```sql
SET hash_mem_multiplier = 1;

SET
```

Включим вывод сообщений в журнал о создании временных файлов:
```sql
SET log_temp_files = 0;

SET
```
Такая настройка означает логировать в журнал создание временных файлов, если их размер больше чем `0 KB`.
Можно поставить размер больше, если маленькие файлы нас не интересуют.

```sql
EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE) SELECT b.book_ref FROM bookings b JOIN tickets t ON b.book_ref = t.book_ref;

                            QUERY PLAN                            
------------------------------------------------------------------
 Hash Join (actual rows=2949857 loops=1)
   Hash Cond: (t.book_ref = b.book_ref)
   ->  Seq Scan on tickets t (actual rows=2949857 loops=1)
   ->  Hash (actual rows=2111110 loops=1)
         Buckets: 1048576  Batches: 4  Memory Usage: 28291kB
         ->  Seq Scan on bookings b (actual rows=2111110 loops=1)
 Planning Time: 12.168 ms
 Execution Time: 3297.194 ms
(8 rows)
```

Читается внутренний набор данных `bookings`, и разбивается на `4` пакета.
Хеш-таблица для первого пакета остается в памяти.
Остальные `3` пакета записываются на диск во временные файлы.

Читается внешний набор данных `tickets`.
Если строка принадлежить первому пакету, она сопоставляется с хеш-таблицей.
Иначе строка записывается на диск в соответствующий пакет.

Т.о. создаются `6` файлов: `3` для `bookings` и `3` для `tickets`.
Количество файлов вычисляется по формуле: `2 * (N - 1)`, где `N` - число пакетов.
Посмотрим сообщения о временных файлах в журнале:
```bash
tail -n 12 /var/lib/postgresql/data/log/postgresql-2024-10-21_011020.log

2024-10-21 01:32:15.151 UTC [54] LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp54.82", size 14257782
2024-10-21 01:32:15.151 UTC [54] STATEMENT:  EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE) SELECT b.book_ref FROM bookings b JOIN tickets t ON b.book_ref = t.book_ref;
2024-10-21 01:32:15.412 UTC [54] LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp54.84", size 19925217
2024-10-21 01:32:15.412 UTC [54] STATEMENT:  EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE) SELECT b.book_ref FROM bookings b JOIN tickets t ON b.book_ref = t.book_ref;
2024-10-21 01:32:15.523 UTC [54] LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp54.81", size 14265342
2024-10-21 01:32:15.523 UTC [54] STATEMENT:  EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE) SELECT b.book_ref FROM bookings b JOIN tickets t ON b.book_ref = t.book_ref;
2024-10-21 01:32:15.789 UTC [54] LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp54.83", size 19938042
2024-10-21 01:32:15.789 UTC [54] STATEMENT:  EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE) SELECT b.book_ref FROM bookings b JOIN tickets t ON b.book_ref = t.book_ref;
2024-10-21 01:32:15.879 UTC [54] LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp54.80", size 14228595
2024-10-21 01:32:15.879 UTC [54] STATEMENT:  EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE) SELECT b.book_ref FROM bookings b JOIN tickets t ON b.book_ref = t.book_ref;
2024-10-21 01:32:16.128 UTC [54] LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp54.85", size 19888605
2024-10-21 01:32:16.128 UTC [54] STATEMENT:  EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE) SELECT b.book_ref FROM bookings b JOIN tickets t ON b.book_ref = t.book_ref;
```

`3` файла размером `14 257 782` относится к первой таблице, и `3` размером `19 938 042` ко второй.

Далее обрабатывается второй пакет.
Из временного файла `base/pgsql_tmp/pgsql_tmp54.80` в хеш-таблицу читаются строки внутреннего набора.
Затем из другого временного файла `base/pgsql_tmp/pgsql_tmp54.83` читаются строки внешнего набора и сопоставляются с хеш-таблицей.

Затем третий пакет и четвертый.
После обработки последнего пакета соединение завершено и временные файлы освобождаются.
Т.о. при нехватке оперативной памяти алгоритм соединения становится двухпроходным.
Обязанность автора запроса - чтобы в хеш-таблицу попали только действительно нужные поля.

Информацию об использовании временных файлов и вообще о вводе-выводе может показать и сама команда `EXPLAIN` с указанием `BUFFERS`:
```sql
EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE, BUFFERS) SELECT b.book_ref FROM bookings b JOIN tickets t ON b.book_ref = t.book_ref;

                             QUERY PLAN                              
---------------------------------------------------------------------
 Hash Join (actual rows=2949857 loops=1)
   Hash Cond: (t.book_ref = b.book_ref)
   Buffers: shared hit=384 read=62478, temp read=12515 written=12515
   ->  Seq Scan on tickets t (actual rows=2949857 loops=1)
         Buffers: shared hit=192 read=49223
   ->  Hash (actual rows=2111110 loops=1)
         Buckets: 1048576  Batches: 4  Memory Usage: 28291kB
         Buffers: shared hit=192 read=13255, temp written=5217
         ->  Seq Scan on bookings b (actual rows=2111110 loops=1)
               Buffers: shared hit=192 read=13255
 Planning:
   Buffers: shared hit=8
 Planning Time: 0.323 ms
 Execution Time: 3037.419 ms
(14 rows)
```

Потребовалось `4` пакета (`Batches: 4`).
Узел `Hash` записывает страницы пакетов во временные файлы `temp written` при построении хеш-таблицы.
Узел `Hash Join` и записывает `temp read`, и читает `written` временные файлы на этапе соединения.


## Соединение нескольких таблиц

Пример плана с двумя соединениями хешированием (имена пассажиров и рейсы):
```sql
EXPLAIN (COSTS OFF) SELECT t.passenger_name, f.flight_no FROM tickets t
JOIN ticket_flights tf ON tf.ticket_no = t.ticket_no
JOIN flights f ON f.flight_id = tf.flight_id;
```

Сначала соединяются билеты (tickets) и перелеты (ticket_flights), причем хеш-таблица строится по таблице билетов.
Затем получившийся набор строк соединяется с перелетами (flights), по которым строится другая хеш-таблица.


Соединение хешированием требует подготовки.
Надо построить хеш таблицу.

Эффективно для больших выборок.

Зависит от порядка соединения.
Внутренний набор должен быть меньше внешнего, чтобы минимизировать хеш-таблицу.

Поддерживает только эквисоединения.
Для хеш-кодов операторы "больше" и "меньше" не имеют смысла.
