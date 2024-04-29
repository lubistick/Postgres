# Низкий уровень

Объект представлен несколькими слоями.

Слой состоит из одного или нескольких файлов-сегментов.

Каждый файл разбит на страницы (они помещаются в буферный кеш при работе).

Для длинных версий строк используется TOAST.

## Расположение файлов

Посмотрим на файлы, принадлежащие таблице:
```sql
CREATE DATABASE data_lowlevel;

CREATE DATABASE
```

```sql
\c data_lowlevel

You are now connected to database "data_lowlevel" as user "postgres".
```

```sql
CREATE TABLE t(id serial PRIMARY KEY, n numeric);

CREATE TABLE
```

Вставим `10 000` строк:
```sql
INSERT INTO t(n) SELECT g.id FROM generate_series(1, 10000) AS g(id);

INSERT 0 10000
```

Базовое имя файла относительно PGDATA можно получить функцией:
```sql
SELECT pg_relation_filepath('t');

 pg_relation_filepath 
----------------------
 base/82127/82129
(1 row)
```

Поскольку таблица находится в табличном пространстве `pg_default`, имя начинается на `base`.
Затем идет каталог для БД:

```sql
SELECT oid FROM pg_database WHERE datname = 'data_lowlevel';

  oid  
-------
 82127
(1 row)
```

Затем собственно имя файла. Его можно узнать следующим образом:
```sql
SELECT relfilenode FROM pg_class WHERE relname = 't';

 relfilenode 
-------------
       82129
(1 row)
```

Тем и удобна функция, что выдает готовый путь без необходимости выполнять несколько запросов к системному каталогу.

Посмотрим на файлы:
```bash
ls -l /var/lib/postgresql/data/base/82127/82129*

-rw-------    1 postgres postgres    450560 Apr 27 12:25 /var/lib/postgresql/data/base/82127/82129
-rw-------    1 postgres postgres     24576 Apr 27 12:25 /var/lib/postgresql/data/base/82127/82129_fsm
-rw-------    1 postgres postgres      8192 Apr 27 12:25 /var/lib/postgresql/data/base/82127/82129_vm
```

Мы видим 3 слоя:
- основной
- карта свободного пространства (`fsm`)
- карта видимости (`vm`)

Аналогично можно посмотреть на файлы индекса:
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

```sql
SELECT pg_relation_filepath('t_pkey');

 pg_relation_filepath 
----------------------
 base/82127/82135
(1 row)
```

```bash
ls -l /var/lib/postgresql/data/base/82127/82135*

-rw-------    1 postgres postgres    245760 Apr 27 12:25 /var/lib/postgresql/data/base/82127/82135
```

И на файлы последовательности:
```sql
SELECT pg_relation_filepath('t_id_seq');

 pg_relation_filepath 
----------------------
 base/82127/82128
(1 row)
```

```bash
ls -l /var/lib/postgresql/data/base/82127/82128*

-rw-------    1 postgres postgres      8192 Apr 27 12:25 /var/lib/postgresql/data/base/82127/82128
```


## Размер объектов и слоев

Размер каждого из файлов, входящих в слой, можно, конечно, посмотреть в файловой системе, но существуют и специальные функции.

Размер каждого слоя в отдельности:
```sql
SELECT pg_relation_size('t', 'main') main, pg_relation_size('t', 'fsm') fsm, pg_relation_size('t', 'vm') vm;

  main  |  fsm  |  vm  
--------+-------+------
 450560 | 24576 | 8192
(1 row)
```

Размер таблицы без индексов (сумма всех слоев):
```sql
SELECT pg_table_size('t');

 pg_table_size 
---------------
        491520
(1 row)
```

Размер индексов таблицы:
```sql
SELECT pg_indexes_size('t');

 pg_indexes_size 
-----------------
          245760
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

```bash
echo $PGDATA

/var/lib/postgresql/data
```

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





