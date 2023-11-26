# Очистка

Периодические задачи:
- удаление старых строк и индексов, которые на них ссылаются
- обновление карты видимости
- обновление карты свободного пространства
- обновление статистики

## Автоочистка

Выполнение описанных периодических задач берет на себя фоновый процесс автоочистки.
Он динамически реагирует на частоту обновления таблиц: чем активней изменения, тем чаще таблица будет обрабатываться.

Посмотрим текущие процессы:
```bash
ps

PID   USER     TIME  COMMAND
    1 postgres  0:00 postgres
   23 postgres  0:00 postgres: checkpointer
   24 postgres  0:00 postgres: background writer
   25 postgres  0:00 postgres: walwriter
   26 postgres  0:00 postgres: autovacuum launcher
   27 postgres  0:00 postgres: stats collector
   28 postgres  0:00 postgres: logical replication launcher
   29 root      0:00 sh
   35 root      0:00 ps
```

- `autovacuum launcher` - планирует работу очистки и запускает необходимое число рабочих процессов, работающих параллельно
- `autovacuum worker` - очищает таблицы отдельной базы данных, требующие обработки

Поговорим подробнее про проблемы, которые решает очистка.

### Старые версии строк и индексов

Из-за механизма многоверсионности в табличных страницах накапливаются старые версии строк, а в страницах индексов - ссылки на эти версии.
Со временем не остается ни одного снимка данных, которому требовались бы старые версии строк.
Такие версии называются "мертвыми".

Очистка удаляет:

- мертвые версии строк из таблиц
- записи из индексов, ссылающиеся на мертвые версии строк таблиц


### Карта видимости

Это служебная информация для таблиц.
В ней отмечены страницы, которые содержат только актуальные версии строк, видимые во всех снимках данных.
Применяется для:

- для оптимизации работы процесса очистки
- для оптимизации сканирования только индекса (в индексе нет информации о версионности, если табличная страница отмечена в карте видимости, то видимость можно не проверять).

Очистка поддерживает карту видимости в актуальном состоянии.

### Карта свободного пространства

Это служебная информация для таблиц и индексов. Используется при вставке новых версий строк.
В ней содержится информация о свободном месте внутри страниц. 
При добавлении новых версий строк место уменьшается, при очистке - увеличивается.

Очистка обновляет карту видимости.

### Статистика

Для работы оптимизатора запросов необходима статистика:
- количество строк в таблицах
- распределение данных в столбцах

При сборе статистики читается случайная выборка данных определенного размера.
Это позволяет быстро собрать информацию даже по очень большим таблицам.

Очистка обновляет статистику.


## Запуск вручную

Очистку и анализ можно запустить вручную, с помощью команд `VACUUM` (только очистка), `ANALYZE` (только анализ) и `VACUUM ANALYZE` (очистка и анализ).

Создадим таблицу, отключив для нее автоматическую очистку:
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

Сейчас в таблице `100 000` строк, все строки имеют ровно `1` версию.

Обновим одну строку:
```sql
UPDATE bloat SET d = d + interval '1 day' WHERE id = 1;

UPDATE 1
```

Запустим очистку вручную с выводом подробностей:
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
После выполнения команды:
- из таблицы вычищена 1 версия
- из индекса удалена 1 запись


## Разрастание размера таблиц и индексов

Очистка не уменьшает размер таблиц и индексов.
"Дыры" в страницах используются для новых данных, но место не возвращается операционной системе.

Негативное влияние:
- перерасход места на диске
- замедление последовательного сканирования таблиц
- уменьшение эффективности индексного доступа

Причины разрастания:
- массовое изменение данных
- неправильная настройка автоочистки
- долгие (висящие) транзакции


### Оценка разрастания таблиц и индексов

Понять, насколько критично разрослись объекты и пора ли предпринимать радикальные меры, можно используя расширение `pgstattuple`.

Установим его:
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
- `tuple_percent` - доля полезной информации, `90 %` - хорошая оценка, она не будет `100 %` из-за накладных расходов

Проверим состояние индекса:
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
- `leaf_pages` - количество листовых страниц
- `avg_leaf_density` - заполненность листовых страниц
- `leaf_fragmentation` - характеристика физической упорядоченности страниц

Теперь обновим сразу половину строк:
```sql
UPDATE bloat SET d = d + interval '1 day' WHERE id % 2 = 0;

UPDATE 50000
```

Посмотрим на таблицу:
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

- увеличился размер таблицы с `4.4 MB` до `6.6 MB`
- уменьшилась плотность информации с `90 %` до `60 %`

Посмотрим на индекс:
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

- заполненность листовых страниц осталась на прежнем уровне `87-89 %`
- количество страниц увеличилось с `85` до `130`

Чтобы не читать всю таблицу целиком, можно выводить приблизительную информацию с помощью `pgstattuple_approx`:
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


## Перестроение индекса

Команда `REINDEX` перестраивает индекс, тем самым уменьшает его физический размер.
При этом полностью блокирует работу с индексом и изменение таблицы.

С опцией `CONCURRENTLY` не блокирует таблицу на запись, но выполняется дольше и может завершиться неудачно:
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

Количество листовых страниц (`85`) и плотность (`89 %`) вернулись к начальным значениям.


## Перестроение таблицы

`VACUUM FULL` - полная очистка. Перестраивает таблицу и ее индексы, уменьшая их физический размер.
Пока идет перестроение, таблица блокируется:

```sql
VACUUM FULL bloat;

VACUUM
```


Посмотрим состояние таблицы:
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

Плотность увеличилась до `90 %`. Освобожденное место отдано операционной системе.


## Обновление большого количества данных без чрезмерного разрастания таблицы

В примере сравним обновление большого массива данных с очисткой и без.

### Отключение автоочистки для всего кластера

Найдем процесс автоочистки и отключим его:
```sql
SELECT pid, backend_start, backend_type FROM pg_stat_activity WHERE backend_type = 'autovacuum launcher';

 pid |        backend_start         |    backend_type     
-----+------------------------------+---------------------
  26 | 2023-11-25 19:00:58.45924+00 | autovacuum launcher
(1 row)
```
```sql
ALTER SYSTEM SET autovacuum = off;

ALTER SYSTEM
```
```sql
SELECT pg_reload_conf();

 pg_reload_conf 
----------------
 t
(1 row)
```

Убедимся, что процесс не работает:
```sql
SELECT pid, backend_start, backend_type FROM pg_stat_activity WHERE backend_type = 'autovacuum launcher';

 pid | backend_start | backend_type 
-----+---------------+--------------
(0 rows)
```


### База данных, таблица и индекс

Создадим базу данных и подключимся к ней:
```sql
CREATE DATABASE admin_maintenance;

CREATE DATABASE
```
```sql
\c admin_maintenance

You are now connected to database "admin_maintenance" as user "postgres".
```

Создадим таблицу с одним числовым столбцом:
```sql
CREATE TABLE t(n numeric);

CREATE TABLE
```

Создадим индекс:
```sql
CREATE INDEX t_n on t(n);

CREATE INDEX
```

Вставим в таблицу `100 000` строк со случайными числами:
```sql
INSERT INTO t SELECT random() FROM generate_series(1,100000);

INSERT 0 100000
```


### Вычисление размера таблицы и индекса

Запишем запрос, который вычисляет размер таблицы и индекса, в переменную `SIZE`:
```sql
\set SIZE 'SELECT pg_size_pretty(pg_table_size(''t'')) table_size, pg_size_pretty(pg_indexes_size(''t'')) index_size \\g (footer=off)'
```

Посмотрим переменную:
```sql
:SIZE

 table_size | index_size 
------------+------------
 4360 kB    | 4256 kB
```


### Изменение строк без очистки

Обновим примерно половину строк:
```sql
UPDATE t SET n = n WHERE n < 0.5;

UPDATE 49742
```

```sql
:SIZE

 table_size | index_size 
------------+------------
 6512 kB    | 6456 kB
```

Размер таблицы и индекса увеличился.

```sql
UPDATE t SET n = n WHERE n < 0.5;

UPDATE 49742
```

```sql
:SIZE

 table_size | index_size 
------------+------------
 8664 kB    | 7496 kB
```

Размер снова увеличился.

```sql
UPDATE t SET n = n WHERE n < 0.5;

UPDATE 49742
```

```sql
:SIZE

 table_size | index_size 
------------+------------
 11 MB      | 10 MB
```

Размер таблицы и индекса постоянно растет.


### Полная очистка

```sql
VACUUM FULL t;

VACUUM
```

```sql
:SIZE

 table_size | index_size 
------------+------------
 4336 kB    | 3104 kB
```

Размер таблицы практически вернулся к начальному.

Индекс стал компактнее - построить индекс по большому объему данных эффективнее, чем добавлять эти данные к индексу построчно.


### Изменение строк с очисткой

Теперь после каждого изменения будем вызывать обычную очистку и сравним результаты:

```sql
UPDATE t SET n = n WHERE n < 0.5;

UPDATE 49742
```

```sql
VACUUM t;

VACUUM
```

```sql
:SIZE

 table_size | index_size 
------------+------------
 6512 kB    | 4640 kB
```
Помним, что `VACUUM` удаляет старые версии строк, но освобожденное место не возвращается операционной системе.
Поэтому размер вырос.

```sql
UPDATE t SET n = n WHERE n < 0.5;

UPDATE 49742
```

```sql
VACUUM t;

VACUUM
```

```sql
:SIZE

 table_size | index_size 
------------+------------
 6512 kB    | 4640 kB
```

И еще один апдейт:
```sql
UPDATE t SET n = n WHERE n < 0.5;

UPDATE 49742
```

```sql
VACUUM t;

VACUUM
```

```sql
:SIZE

 table_size | index_size 
------------+------------
 6512 kB    | 4640 kB
```

Размер увеличился один раз, затем стабилизировался.

### Итоги

Пример показывает, что изменение большого объема данных по возможности следует разделить на несколько транзакций.
Это позволит автоматической очистке своевременно удалять ненужные версии строк, что позволит избежать чрезмерного разрастания таблицы.

Восстанавливаем автоочистку.
```sql
ALTER SYSTEM RESET autovacuum;

ALTER SYSTEM
```

```sql
SELECT pg_reload_conf();

 pg_reload_conf 
----------------
 t
(1 row)
```
