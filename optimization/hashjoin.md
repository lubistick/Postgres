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


Теперь уменьшим work_mem так, чтобы хеш-таблица не поместилась, и включим вывод сообщений о временных файлах:
```sql
SET work_mem = '32MB';
```

```sql
SET log_temp_files = 0;
```
Если в нашем сеансе будут создаваться временные файлы размером больше, чем 0 байт, значит информацию о них нужно записывать в журнал.
Можно поставить размер больше, если маленькие файлы нас не интересуют.
Но нас в принципе интересует, какие файлы создавались.

```sql
EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE) SELECT b.book_ref FROM bookings b JOIN tickets t ON b.book_ref = t.book_ref; 
```

Время выполнения запроса возросло.
Потребовалось 4 пакета: один из них сразу же загружается в оперативную память, а для 3-х других создадутся файлы на диске.
Т.е. 3 файла, которые потом будут загружаться в хеш-таблицу и еще 3 файла для второго набора данных.

Посмотрим сообщения о временных файлах в журнале - поскольку были использованы 4 пакета, то файлов будет (4 - 1) * 2 = 6:
```bash
tail -n 18 /var/log/postgresql/postgresql-10-main.log (исправить путь)
```

По размеру видно, что часть файлов относится к первой таблице (они меньше), а часть - ко второй (они больше).

Информацию об использовании временных файлов и вообще о вводе-выводе может показать и сама команда EXPLAIN с указанием BUFFERS:
```sql
EXPLAIN (COSTS OFF, TIMING OFF, ANALYZE, BUFFERS) SELECT b.book_ref FROM bookings b JOIN tickets t ON b.book_ref = t.book_ref; 
```

- Hash: temp written - страницы пакетов, записанные при построении хеш-таблицы.
- Hash Join: temp read/written - чтение и запись временных файлов на этапе соединения.


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
