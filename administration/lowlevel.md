# Низкий уровень

Каждый объект БД представлен несколькими слоями.
Слой состоит из одного или нескольких файлов-сегментов.
Сегменты разбиты на страницы, которые попадают в [буферный кеш](wal.md) при работе.


## Расположение слоев

Создадим БД и подключимся к ней:
```sql
CREATE DATABASE data_lowlevel;

CREATE DATABASE
```

```sql
\c data_lowlevel

You are now connected to database "data_lowlevel" as user "postgres".
```

Создадим таблицу `t` и вставим `10 000` строк:
```sql
CREATE TABLE t(id serial PRIMARY KEY, n numeric);

CREATE TABLE
```

```sql
INSERT INTO t(n) SELECT g.id FROM generate_series(1, 10000) AS g(id);

INSERT 0 10000
```


### Слои таблицы

Получим базовое имя слоев таблицы относительно каталога `PGDATA`:
```sql
SELECT pg_relation_filepath('t');

 pg_relation_filepath 
----------------------
 base/82127/82129
(1 row)
```

Посмотрим переменную `PGDATA`:
```bash
echo $PGDATA

/var/lib/postgresql/data
```

Посмотрим на слои:

```bash
ls -l /var/lib/postgresql/data/base/82127/82129*

-rw-------    1 postgres postgres    450560 Apr 27 12:25 /var/lib/postgresql/data/base/82127/82129
-rw-------    1 postgres postgres     24576 Apr 27 12:25 /var/lib/postgresql/data/base/82127/82129_fsm
-rw-------    1 postgres postgres      8192 Apr 27 12:25 /var/lib/postgresql/data/base/82127/82129_vm
```

- основной
- карта свободного пространства (`_fsm`)
- карта видимости (`_vm`)

Посмотрим с помощью функции `pg_relation_size` размер слоев:
```sql
SELECT
  pg_relation_size('t', 'main') main,
  pg_relation_size('t', 'fsm') fsm,
  pg_relation_size('t', 'vm') vm
;

  main  |  fsm  |  vm  
--------+-------+------
 450560 | 24576 | 8192
(1 row)
```


### Слои индекса

Посмотрим информацию о таблице:
```sql
\d t

                            Table "public.t"
 Column |  Type   | Collation | Nullable |            Default
--------+---------+-----------+----------+-------------------------------
 id     | integer |           | not null | nextval('t_id_seq'::regclass)
 n      | numeric |           |          |
Indexes:
    "t_pkey" PRIMARY KEY, btree (id)
```

Посмотрим базовое имя слоев индекса относительно `PGDATA`:
```sql
SELECT pg_relation_filepath('t_pkey');

 pg_relation_filepath 
----------------------
 base/82127/82135
(1 row)
```

Посмотрим на слои:
```bash
ls -l /var/lib/postgresql/data/base/82127/82135*

-rw-------    1 postgres postgres    245760 Apr 27 12:25 /var/lib/postgresql/data/base/82127/82135
```

Посмотрим размер индексов таблицы с помощью функции `pg_indexes_size`:
```sql
SELECT pg_indexes_size('t');

 pg_indexes_size 
-----------------
          245760
(1 row)
```


### Слои последовательности

Посмотрим базовое имя слоев последовательности относительно `PGDATA`:
```sql
SELECT pg_relation_filepath('t_id_seq');

 pg_relation_filepath 
----------------------
 base/82127/82128
(1 row)
```

Посмотрим на слои:
```bash
ls -l /var/lib/postgresql/data/base/82127/82128*

-rw-------    1 postgres postgres      8192 Apr 27 12:25 /var/lib/postgresql/data/base/82127/82128
```


## Размер объектов

Размер таблицы без индексов (сумма всех слоев):
```sql
SELECT pg_table_size('t');

 pg_table_size 
---------------
        491520
(1 row)
```

И размер всей таблицы, включая индексы:
```sql
SELECT pg_total_relation_size('t');

 pg_total_relation_size 
------------------------
                 737280
(1 row)
```


## TOAST

Любая версия строки в `Postgres` должна целиком помещаться на одну страницу.
Для "длинных" версий строк применяется технология TOAST - The Oversized Attribute Storage Technique.

В таблице `t` есть столбец типа `numeric`.
Этот тип может работать с очень большими числами. Например, с такими:
```sql
SELECT length((123456789::numeric ^ 12345::numeric)::text);

 length 
--------
  99907
(1 row)
```

При этом, если вставить такое значение в таблицу, размер файлов не изменится:
```sql
SELECT pg_relation_size('t', 'main');

 pg_relation_size 
------------------
           450560
(1 row)
```

```sql
INSERT INTO t(n) SELECT 123456789::numeric ^ 12345::numeric;

INSERT 0 1
```

```sql
SELECT pg_relation_size('t', 'main');

 pg_relation_size 
------------------
           450560
(1 row)
```

Поскольку версия строки не помещается на одну страницу, она храниться в отдельной TOAST-таблице.
TOAST-таблица и индекс к ней создаются автоматически для каждой таблицы, в которой есть потенциально "длинный" тип данных, и используется по необходимости.

Имя и идентификатор такой таблицы можно найти следующим образом:
```sql
SELECT relname, relfilenode FROM pg_class WHERE oid = (
  SELECT reltoastrelid FROM pg_class WHERE relname = 't'
);

    relname     | relfilenode 
----------------+-------------
 pg_toast_82129 |       82133
(1 row)
```

Вот и файлы TOAST-таблицы:

```bash
ls -l /var/lib/postgresql/data/base/82127/82133*

-rw-------    1 postgres postgres     57344 Apr 28 20:04 /var/lib/postgresql/data/base/82127/82133
-rw-------    1 postgres postgres     24576 Apr 28 20:00 /var/lib/postgresql/data/base/82127/82133_fsm
```

Существуют несколько стратегий работы с длинными значениями.
Название стратегии указывается в поле `Storage`:
```sql
\d+ t

                                                       Table "public.t"
 Column |  Type   | Collation | Nullable |            Default            | Storage | Compression | Stats target | Description
--------+---------+-----------+----------+-------------------------------+---------+-------------+--------------+-------------
 id     | integer |           | not null | nextval('t_id_seq'::regclass) | plain   |             |              |
 n      | numeric |           |          |                               | main    |             |              |
Indexes:
    "t_pkey" PRIMARY KEY, btree (id)
Access method: heap
```

- `plain` - TOAST не применяется, тип имеет фиксированную длину
- `main` - приоритет сжатия
- `extended` - применяется как сжатие, так и отдельное хранение
- `external` - только отдельное хранение без сжатия

Стратегию можно изменить, если это необходимо.
Например, если известно, что в столбце хранятся уже сжатые данные, разумно поставить стратегию `external`.

Просто для примера:
```sql
ALTER TABLE t ALTER COLUMN n SET STORAGE extended;

ALTER TABLE
```

Эта операция не меняет существующие данные в таблице, но определяет стратегию работы с новыми данными.



ПРАКТИКА

Нежурналируемая таблица

```sql
CREATE TABLESPACE ts LOCATION '/var/lib/postgresql/ts_dir';

CREATE TABLESPACE
```

```sql
CREATE UNLOGGED TABLE u(n integer) TABLESPACE ts;

CREATE TABLE
```

```sql
INSERT INTO u(n) SELECT n FROM generate_series(1, 1000) n;

INSERT 0 1000
```

```sql
SELECT pg_relation_filepath('u');

            pg_relation_filepath             
---------------------------------------------
 pg_tblspc/90320/PG_14_202107181/82127/90321
(1 row)
```

Посмотрим на файлы таблицы.

Для таблицы существует слой `init`:

```bash
ls -l /var/lib/postgresql/data/pg_tblspc/90320/PG_14_202107181/82127/90321*

-rw-------    1 postgres postgres     40960 Apr 29 11:35 /var/lib/postgresql/data/pg_tblspc/90320/PG_14_202107181/82127/90321
-rw-------    1 postgres postgres     24576 Apr 29 11:35 /var/lib/postgresql/data/pg_tblspc/90320/PG_14_202107181/82127/90321_fsm
-rw-------    1 postgres postgres         0 Apr 29 11:33 /var/lib/postgresql/data/pg_tblspc/90320/PG_14_202107181/82127/90321_init
```

Удалим созданное ТП:
```sql
DROP TABLE u;

DROP TABLE
```

```sql
DROP TABLESPACE ts;

DROP TABLESPACE
```


Работа с текстовым столбцом.

```sql
CREATE TABLE t(s text);

CREATE TABLE
```

```sql
\d+ t

                                           Table "public.t"
 Column | Type | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
--------+------+-----------+----------+---------+----------+-------------+--------------+-------------
 s      | text |           |          |         | extended |             |              |
Access method: heap
```

По умолчанию для типа `text` используется стратегия `extended`.
Изменим стратегию на `external`:

```sql
ALTER TABLE t ALTER COLUMN s SET STORAGE external;

ALTER TABLE
```

```sql
INSERT INTO t(s) VALUES ('Короткая строка.');

INSERT 0 1
```

```sql
INSERT INTO t(s) VALUES (repeat('А', 3456));

INSERT 0 1
```

Проверим TOAST-таблицу:
```sql
SELECT relname FROM pg_class WHERE oid = (SELECT reltoastrelid FROM pg_class WHERE relname = 't');

    relname     
----------------
 pg_toast_90324
(1 row)
```

TOAST-таблица "спрятана", т.к. находится в схеме, которой нет в пути поиска.
И это правильно, поскольку TOAST работает прозрачно для пользователя.
Но заглянуть в таблицу все-таки можно:

```sql
SELECT chunk_id, chunk_seq, length(chunk_data) FROM pg_toast.pg_toast_90324 ORDER BY chunk_id, chunk_seq;

 chunk_id | chunk_seq | length 
----------+-----------+--------
    90329 |         0 |   1996
    90329 |         1 |   1996
    90329 |         2 |   1996
    90329 |         3 |    924
(4 rows)
```

Повторяющаяся буква `А` русская, поэтому занимает `2` байта: `(1996 + 1996 + 1996 + 924) / 2 = 3456`

Видно, что в TOAST таблицу попала только одна длинная строка (общий размер совпадает с длиной строки).
Короткая строка не вынесена в TOAST просто потому, что в этом нет необходимости - версия строки и без этого помещается в страницу.


Сравнение размеров БД и таблиц в ней

```sql
CREATE DATABASE data_lowlevel;

CREATE DATABASE
```

```sql
\c data_lowlevel

You are now connected to database "data_lowlevel" as user "postgres".
```

Даже пустая БД содержит таблицы, относящиеся к системному каталогу.
Полный список отношений можно получить из таблицы `pg_class`.
Из выборки надо исключить:
- таблицы, общие для всего кластера (они не относятся к текущей БД)
- индексы и TOAST-таблицы (они будут автоматически учтены при подсчете размера)

```sql
SELECT sum(pg_total_relation_size(oid)) FROM pg_class WHERE NOT relisshared AND relkind = 'r';

   sum   
---------
 8626176
(1 row)
```

Размер БД оказывается несколько больше:
```sql
SELECT pg_database_size('data_lowlevel');

 pg_database_size 
------------------
          8782627
(1 row)
```

Дело в том, что функция `pg_database_size` возвращает размер каталога файловой системы, а в этом каталоге находятся несколько служебных файлов.

```sql
SELECT oid FROM pg_database WHERE datname = 'data_lowlevel';

  oid  
-------
 98511
(1 row)
```

```bash
ls -l /var/lib/postgresql/data/base/98511/[^0-9]*

-rw-------    1 postgres postgres         3 Apr 29 15:16 /var/lib/postgresql/data/base/98511/PG_VERSION
-rw-------    1 postgres postgres       512 Apr 29 15:16 /var/lib/postgresql/data/base/98511/pg_filenode.map
-rw-------    1 postgres postgres    155936 Apr 29 15:16 /var/lib/postgresql/data/base/98511/pg_internal.init
```

- `pg_filenode.map` - отображение `oid` некоторых таблиц и имена файлов
- `pg_internal.init` - кеш системного каталога
- `PG_VERSION` - версия `Postgres`

Из-за того, что одни функции работают на уровне объектов БД, а другие на уровне файловой системы, бывает сложно точно сопоставить возвращаемые размеры.
Это относится и к функции `pg_tablespace_size`.

