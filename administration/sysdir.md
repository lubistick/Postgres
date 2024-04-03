# Системный каталог

В схеме `pg_catalog` располагаются таблицы и представления, которые описывают все объекты кластера баз данных.

Стандартом SQL регламентировано само понятие системного каталога и каким образом с ним нужно работать.
Postgres поддерживает стандарт, поэтому к системному каталогу можно обратиться и через схему `information_schema.`

Схема `pg_catalog` есть в каждой БД кластера.


## Некоторые объекты системного каталога

Создадим БД и текстовые объекты:

```sql
CREATE DATABASE data_catalog;

CREATE DATABASE
```

```sql
\c data_catalog

You are now connected to database "data_catalog" as user "postgres".
```

```sql
CREATE TABLE employees(
  id serial PRIMARY KEY,
  name text,
  manager integer
);

CREATE TABLE
```

```sql
CREATE VIEW top_manager AS
SELECT * FROM employees WHERE manager IS NULL;

CREATE VIEW
```

Некоторые таблицы системного каталога нам уже знакомы из предыдущей темы. Это базы данных:

```sql
SELECT * FROM pg_database WHERE datname = 'data_catalog' \gx

-[ RECORD 1 ]-+-------------
oid           | 65720
datname       | data_catalog
datdba        | 10
encoding      | 6
datcollate    | en_US.utf8
datctype      | en_US.utf8
datistemplate | f
datallowconn  | t
datconnlimit  | -1
datlastsysoid | 13776
datfrozenxid  | 727
datminmxid    | 1
dattablespace | 1663
datacl        |
```

И схемы:

```sql
SELECT * FROM pg_namespace WHERE nspname = 'public' \gx

-[ RECORD 1 ]---------------------------------
oid      | 2200
nspname  | public
nspowner | 10
nspacl   | {postgres=UC/postgres,=UC/postgres}
```

Важная таблица `pg_class` хранит описание целого ряда объектов: таблиц, представлений (включая материализованные), индексов, последовательностей.
Все эти объекты называются в PostgreSQL общим словом "отношение" (relation), отсюда префикс "rel" в названии столбцов:

```sql
SELECT relname, relkind, relnamespace, relfilenode, relowner, reltablespace  FROM pg_class WHERE relname ~ '^(emp|top).*';

     relname      | relkind | relnamespace | relfilenode | relowner | reltablespace 
------------------+---------+--------------+-------------+----------+---------------
 employees_id_seq | S       |         2200 |       65721 |       10 |             0
 employees        | r       |         2200 |       65722 |       10 |             0
 employees_pkey   | i       |         2200 |       65728 |       10 |             0
 top_manager      | v       |         2200 |           0 |       10 |             0
(4 rows)
```
Типы объектов различаются по столбцу `relkind`:
- S - последовательность
- r - таблица
- i - индекс
- v - представление


Конечно, для каждого типа объектов имеет смысл только часть столбцов.
Кроме того, удобнее смотреть не на многочисленные OID, а на нормальные значения.
Для этого существуют различные представления, например:

```sql
SELECT schemaname, tablename, tableowner, tablespace FROM pg_tables WHERE schemaname = 'public';

 schemaname | tablename | tableowner | tablespace 
------------+-----------+------------+------------
 public     | employees | postgres   |
(1 row)
```


```sql
SELECT * FROM pg_views WHERE schemaname = 'public';

 schemaname |  viewname   | viewowner |              definition              
------------+-------------+-----------+--------------------------------------
 public     | top_manager | postgres  |  SELECT employees.id,               +
            |             |           |     employees.name,                 +
            |             |           |     employees.manager               +
            |             |           |    FROM employees                   +
            |             |           |   WHERE (employees.manager IS NULL);
(1 row)
```


## Использование команд psql

Список всех "отношений" можно посмотреть командой `\d*` в `psql`, где `*` - символ, обозначающий тип объекта (как `relkind`).
Например, таблицы:

```sql
\dt

           List of relations
 Schema |   Name    | Type  |  Owner
--------+-----------+-------+----------
 public | employees | table | postgres
(1 row)
```

Или представления:

```sql
\dv

           List of relations
 Schema |    Name     | Type |  Owner
--------+-------------+------+----------
 public | top_manager | view | postgres
(1 row)
```

Эти команды можно снабдить модификатором "+", чтобы получить больше информации:

```sql
\dt+

                                       List of relations
 Schema |   Name    | Type  |  Owner   | Persistence | Access method |    Size    | Description
--------+-----------+-------+----------+-------------+---------------+------------+-------------
 public | employees | table | postgres | permanent   | heap          | 8192 bytes |
(1 row)
```

Чтобы получить детальную информацию о конкретном объекте, надо воспользоваться командой `\d` без дополнительной буквы:

```sql
\d top_manager

             View "public.top_manager"
 Column  |  Type   | Collation | Nullable | Default
---------+---------+-----------+----------+---------
 id      | integer |           |          |
 name    | text    |           |          |
 manager | integer |           |          |
```

Модификатор `+` остается в силе.

```sql
\d+ top_manager

                          View "public.top_manager"
 Column  |  Type   | Collation | Nullable | Default | Storage  | Description
---------+---------+-----------+----------+---------+----------+-------------
 id      | integer |           |          |         | plain    |
 name    | text    |           |          |         | extended |
 manager | integer |           |          |         | plain    |
View definition:
 SELECT employees.id,
    employees.name,
    employees.manager
   FROM employees
  WHERE employees.manager IS NULL;
```

Помимо "отношений", аналогичным образом можно смотреть и на другие объекты, такие как схемы (`\dn`) или функции (`\df`).

Модификатор "S" позволяет вывести не только пользовательские, но и системные объекты.
А с помощью шаблона можно ограничить выборку:

```sql
\dfS pg*size

                                  List of functions
   Schema   |          Name          | Result data type | Argument data types | Type
------------+------------------------+------------------+---------------------+------
 pg_catalog | pg_column_size         | integer          | "any"               | func
 pg_catalog | pg_database_size       | bigint           | name                | func
 pg_catalog | pg_database_size       | bigint           | oid                 | func
 pg_catalog | pg_indexes_size        | bigint           | regclass            | func
 pg_catalog | pg_relation_size       | bigint           | regclass            | func
 pg_catalog | pg_relation_size       | bigint           | regclass, text      | func
 pg_catalog | pg_table_size          | bigint           | regclass            | func
 pg_catalog | pg_tablespace_size     | bigint           | name                | func
 pg_catalog | pg_tablespace_size     | bigint           | oid                 | func
 pg_catalog | pg_total_relation_size | bigint           | regclass            | func
(10 rows)
```

`Psql` предлагает большое количество команд для просмотра системного каталога.
Как правило, они имеют мнемонические имена.
Например, `\df` - describe function, `\sf` - show function:

```sql
\sf pg_catalog.pg_database_size(oid)

CREATE OR REPLACE FUNCTION pg_catalog.pg_database_size(oid)
 RETURNS bigint
 LANGUAGE internal
 PARALLEL SAFE STRICT
AS $function$pg_database_size_oid$function$
```

Полный список всегда можно посмотреть в документации или командой `\?`.


## Изучение структуры системного каталога

Все команды `psql`, описывающие объекты, обращаются к таблицам системного каталога.
Чтобы посмотреть, какие запросы на самом деле выполняет `psql`, можно установить переменную `ECHO_HIDDEN`:










---

Что такое системный каталог и как к нему обращаться

Объекты системного каталога и их расположение

Правила именования объектов

Специальные типы данных

---


SQL-доступ

просмотр: SELECT
изменение: CREATE, ALTER, DROP

Доступ в psql

специальные команды для удобного просмотра

---


OID - тип для идентификатора объекта

первичные и внешние ключи в таблицах системного каталога

скрытый столбец, в запросах надо указывать явно


Reg-типы

псевдонимы OID для некоторых таблиц системного каталога (regclass для pg_class и т.п.)

привидение текстового имени объекта к типу OID и обратно




