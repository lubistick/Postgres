# Очистка

Периодические задачи:

1
Очистка страниц от исторических данных, которые образуются из-за многоверсионности.
Из таблиц вычищаются мертвые версии строк.
Из индексов вычищаются записи, ссылающиеся на мертвые версии.

2
Обновление карты видимости.
Отмечает страницы, на которых все версии строк видны во всех снимках.
Используется для оптимизации работы процесса очистки.
Используется для оптимизации сканирования только индекса (в индексе нет информации о версионности, если страница отмечена в карте видимости, то видимость можно не проверять).
Существует только для таблиц.

3
Обновление карты свободного пространства.
Отмечает свободное пространство в страницах после очистки.
Используется при вставке новых версий строк.
Существует и для таблиц, и для индексов.

4
Обновление статистики.
Используется оптимизатором запросов.
Вычисляется на основе случайной выборки.

5
Заморозка.
Предотвращение последствий переполнения 32-битного счетчика транзакций.



Автоматическая очистка

Autovacuum launcher

Фоновый процесс.
Периодически запускает рабочие процессы.

Autovacuum worker

Очищает таблицы отдельной базы данных, требующие обработки.




Создадим таблицу, в целях эксперимента отключив для нее автоматическую очистку, чтобы контролировать время срабатывания:
```sql
CREATE TABLE bloat(
  id integer GENERATED ALWAYS AS IDENTITY,
  d timestamptz
) WITH (autovacuum_enabled = off);

CREATE TABLE
```

Заполним таблицу данными:
```sql
INSERT INTO bloat(d) SELECT current_timestamp FROM generate_series(1,100000);

INSERT 0 100000
```

Создадим индекс:
```sql
CREATE INDEX ON bloat(d);

CREATE INDEX
```

Сейчас в таблице 100 000 строк, все строки имеют ровно одну версию.

Обновим одну строку:
```sql
UPDATE bloat SET d = d + interval '1 day' WHERE id = 1;

UPDATE 1
```

Запустим очистку вручную и попросим ее рассказать, что происходит:
```sql
VACUUM (verbose) bloat;

INFO:  vacuuming "bookings.bloat"
INFO:  table "bloat": index scan bypassed: 1 pages from table (0.18% of total) have 1 dead item identifiers
INFO:  table "bloat": found 1 removable, 100000 nonremovable row versions in 541 out of 541 pages
DETAIL:  0 dead row versions cannot be removed yet, oldest xmin: 913
Skipped 0 pages due to buffer pins, 0 frozen pages.
CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s.
VACUUM
```
Из вывода команды можно заключить, что:
- из таблицы вычищена 1 версия
- из индекса удалена одна запись (вроде не удалена)



Проблема разрастания

1
Очистка не уменьшает размер таблиц и индексов.
"Дыры" в страницах используются для новых данных, но место не возвращается операционной системе.

2
Причины разрастания.
Неправильная настройка автоочистки.
Массовое изменение данных.
Долгие транзакции.

3
Негативное влияние.
Перерасход места на диске.
Замедление последовательного сканирования таблиц.
Уменьшение эффективности индексного доступа.


## Оценка разрастания таблиц и индексов

Понять, насколько критично разрослись объекты и пора ли предпринимать радикальные меры, можно разными способами:
- запросами к системному каталогу
- используя расширение `pgstattuple`

Устанавливаем расширение:
```sql
CREATE EXTENSION pgstattuple;

CREATE EXTENSION
```

С помощью расширения проверим состояние таблицы:
```sql
SELECT * FROM pgstattuple('bloat') \gx

-[ RECORD 1 ]------+--------
table_len          | 4431872
tuple_count        | 100000
tuple_len          | 4000000
tuple_percent      | 90.26
dead_tuple_count   | 0
dead_tuple_len     | 0
dead_tuple_percent | 0
free_space         | 16720
free_percent       | 0.38
```

Колонки:
- `table_len` - размер таблицы
- `tuple_percent` - доля полезной информации (не 100% из-за накладных расходов)

И состояние индекса:
```sql
SELECT * FROM pgstatindex('bloat_d_idx') \gx

-[ RECORD 1 ]------+-------
version            | 4     
tree_level         | 1     
index_size         | 712704
root_block_no      | 3
internal_pages     | 1
leaf_pages         | 85
empty_pages        | 0
deleted_pages      | 0
avg_leaf_density   | 89.17
leaf_fragmentation | 0
```

Колонки:
- `leaf_pages` - количество листовых страниц индекса
- `avg_leaf_density` - заполненность листовых страниц
- `leaf_fragmentation` - характеристика физической упорядоченности страниц

Теперь обновим сразу половину строк:
```sql
UPDATE bloat SET d = d + interval '1 day' WHERE id % 2 = 0;

UPDATE 50000
```

Посмотрим на таблицу снова:
```sql
SELECT * FROM pgstattuple('bloat') \gx

-[ RECORD 1 ]------+--------
table_len          | 6643712
tuple_count        | 100000
tuple_len          | 4000000
tuple_percent      | 60.21
dead_tuple_count   | 50000
dead_tuple_len     | 2000000
dead_tuple_percent | 30.1
free_space         | 21000
free_percent       | 0.32
```

Увеличился размер таблицы, уменьшилась плотность информации.

Чтобы не читать всю таблицу целиком, можно попросить `pgstattuple` показать приблизительную информацию:
```sql
SELECT * FROM pgstattuple_approx('bloat') \gx

-[ RECORD 1 ]--------+--------------------
table_len            | 6643712
scanned_percent      | 100
approx_tuple_count   | 100000
approx_tuple_len     | 4000000
approx_tuple_percent | 60.207305795314426
dead_tuple_count     | 50000
dead_tuple_len       | 2000000
dead_tuple_percent   | 30.103652897657213
approx_free_space    | 21000
approx_free_percent  | 0.31608835542540076
```

И посмотрим на индекс:
```sql
SELECT * FROM pgstatindex('bloat_d_idx') \gx

-[ RECORD 1 ]------+--------
version            | 4
tree_level         | 1
index_size         | 1081344
root_block_no      | 3
internal_pages     | 1
leaf_pages         | 130
empty_pages        | 0
deleted_pages      | 0
avg_leaf_density   | 87.27
leaf_fragmentation | 0
```

Заполненность листовых страниц осталась на прежнем уровне, но количество страниц увеличилось.





## Перестроение объектов

Полная очистка.
Полностью перестраивает содержимое таблиц и индексов.
Полностью блокирует работу с таблицей.

Перестроение индексов.
Перестраивает индексы.
Полностью блокирует работу с индексом и блокирует изменение таблицы.

Неблокирующее перестроение индексов.
Перестраивает индексы, не блокируя изменения таблицы.
Выполняется дольше и может завершиться неудачно.
Не транзакционно.
Не работает для системных индексов.
Не работает для индексов, связанных с ограничениями-исключениями.

Для перестроения индексов удобно использовать команду REINDEX с указанием CONCURRENTLY.
Это позволяет не останавливать работу системы на время перестроения:
```sql
REINDEX TABLE CONCURRENTLY bloat;

REINDEX
```

Теперь посмотрим на индекс:
```sql
SELECT * FROM pgstatindex('bloat_d_idx') \gx

-[ RECORD 1 ]------+-------
version            | 4
tree_level         | 1
index_size         | 712704
root_block_no      | 3
internal_pages     | 1
leaf_pages         | 85
empty_pages        | 0
deleted_pages      | 0
avg_leaf_density   | 89.17
leaf_fragmentation | 0
```

Количество листовых страниц и плотность вернулись к начальным значениям.

Для перестроения таблицы вместе с ее индексами можно воспользоваться командой VACUUM FULL.
Однако, в отличие от REINDEX .. CONCURRENTLY, она полностью блокирует работу с таблицей:
```sql
VACUUM FULL bloat;

VACUUM
```

```sql
SELECT * FROM pgstattuple('bloat') \gx

-[ RECORD 1 ]------+--------
table_len          | 4431872
tuple_count        | 100000
tuple_len          | 4000000
tuple_percent      | 90.26
dead_tuple_count   | 0
dead_tuple_len     | 0
dead_tuple_percent | 0
free_space         | 16724
free_percent       | 0.38
```

Плотность увеличилась, освобожденное место отдано операционной системе.









