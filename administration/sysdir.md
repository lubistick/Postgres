# Системный каталог

В схеме `pg_catalog` располагаются таблицы и представления, которые описывают все объекты кластера баз данных.
Эта схема есть в каждой БД кластера.
Все таблицы и представления системного каталога начинаются с префикса `pg_`.
Названия столбцов имеют трехбуквенный префикс, который как правило соответствует имени таблицы.

Например, в таблице `pg_tablespace` все столбцы начинаются с префикса `spc`:
```sql
SELECT * FROM pg_tablespace WHERE spcname = 'pg_global';

 oid  |  spcname  | spcowner | spcacl | spcoptions
------+-----------+----------+--------+------------
 1664 | pg_global |       10 |        |
(1 row)
```

В кластере в каждой БД создается свой набор таблиц системного каталога.
Однако существуют несколько объектов каталога, которые являются общими для всего кластера, например, список самих БД.
Эти таблицы хранятся вне какой-либо БД, но при этом одинаково видны из каждой БД.

Стандартом SQL регламентировано понятие системного каталога и каким образом с ним работать.
Postgres поддерживает стандарт, поэтому к системному каталогу можно обратиться и через схему `information_schema`.


## Объекты системного каталога

Создадим тестовую БД, в ней таблицу и представление:

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
CREATE VIEW top_managers AS
SELECT * FROM employees WHERE manager IS NULL;

CREATE VIEW
```


### Базы данных

Информация о базах данных:
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


### Схемы

Информация о схемах:
```sql
SELECT * FROM pg_namespace WHERE nspname = 'public' \gx

-[ RECORD 1 ]---------------------------------
oid      | 2200
nspname  | public
nspowner | 10
nspacl   | {postgres=UC/postgres,=UC/postgres}
```


### Отношения

Важная таблица `pg_class` хранит описание целого ряда объектов: таблиц, представлений (включая материализованные), индексов, последовательностей.
Все эти объекты называются в PostgreSQL общим словом "отношение" (relation), отсюда префикс "rel" в названии столбцов:

```sql
SELECT relname, relkind, relnamespace, relfilenode, relowner, reltablespace  FROM pg_class WHERE relname ~ '^(emp|top).*';

     relname      | relkind | relnamespace | relfilenode | relowner | reltablespace 
------------------+---------+--------------+-------------+----------+---------------
 employees_id_seq | S       |         2200 |       65721 |       10 |             0
 employees        | r       |         2200 |       65722 |       10 |             0
 employees_pkey   | i       |         2200 |       65728 |       10 |             0
 top_managers     | v       |         2200 |           0 |       10 |             0
(4 rows)
```
Типы объектов различаются по столбцу `relkind`:
- `S` - последовательность
- `r` - таблица
- `i` - индекс
- `v` - представление

Для каждого типа объектов имеет смысл только часть столбцов.
Кроме того, удобнее смотреть не на многочисленные OID, а на нормальные значения.
Для этого существуют различные представления, например:


### Таблицы

```sql
SELECT schemaname, tablename, tableowner, tablespace FROM pg_tables WHERE schemaname = 'public';

 schemaname | tablename | tableowner | tablespace 
------------+-----------+------------+------------
 public     | employees | postgres   |
(1 row)
```


### Представления

```sql
SELECT * FROM pg_views WHERE schemaname = 'public';

 schemaname |  viewname    | viewowner |              definition              
------------+--------------+-----------+--------------------------------------
 public     | top_managers | postgres  |  SELECT employees.id,               +
            |              |           |     employees.name,                 +
            |              |           |     employees.manager               +
            |              |           |    FROM employees                   +
            |              |           |   WHERE (employees.manager IS NULL);
(1 row)
```


## Использование команд psql

`psql` предлагает большое количество команд для просмотра системного каталога.
Как правило, они имеют мнемонические имена.
Например, `\df` - describe function, `\sf` - show function.
Полный список всегда можно посмотреть в документации или командой `\?`.

Посмотрим список всех отношений командой `\d`:
```sql
\d

                List of relations
 Schema |       Name        |   Type   |  Owner   
--------+-------------------+----------+----------
 public | employees         | table    | postgres
 public | employees_id_seq  | sequence | postgres
 public | top_managers      | view     | postgres
(3 rows)
```

Модификатор `+` дает больше информации:
```sql
\d+

                                            List of relations
 Schema |       Name       |   Type   |  Owner   | Persistence | Access method |    Size    | Description
--------+------------------+----------+----------+-------------+---------------+------------+-------------
 public | employees        | table    | postgres | permanent   | heap          | 8192 bytes |
 public | employees_id_seq | sequence | postgres | permanent   |               | 8192 bytes |
 public | top_manager      | view     | postgres | permanent   |               | 0 bytes    |
(3 rows)
```

Получим детальную информацию о конкретном объекте, воспользуемся командой `\d` без дополнительной буквы:

```sql
\d top_managers

             View "public.top_managers"
 Column  |  Type   | Collation | Nullable | Default
---------+---------+-----------+----------+---------
 id      | integer |           |          |
 name    | text    |           |          |
 manager | integer |           |          |
```

Модификатор `+` остается в силе:

```sql
\d+ top_managers

                          View "public.top_managers"
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


### Таблицы

Символ `t` отфильтровывает только таблицы:
```sql
\dt

           List of relations
 Schema |   Name    | Type  |  Owner
--------+-----------+-------+----------
 public | employees | table | postgres
(1 row)
```


### Представления

Символ `v`, чтобы видеть только представления:

```sql
\dv

           List of relations
 Schema |    Name      | Type |  Owner
--------+--------------+------+----------
 public | top_managers | view | postgres
(1 row)
```


### Схемы

Создадим временную таблицу:

```sql
CREATE TEMP TABLE t(n integer);

CREATE TABLE
```

Посмотрим на список схем командой `\dn`.
Модификатор `S` выводит не только пользовательские, но и системные объекты.

```sql
\dnS

        List of schemas
        Name        |  Owner
--------------------+----------
 information_schema | postgres
 pg_catalog         | postgres
 pg_temp_4          | postgres
 pg_toast           | postgres
 pg_toast_temp_4    | postgres
 public             | postgres
(6 rows)
```

Временная таблица расположена в схеме `pg_temp_N`, где N - некоторое число.
Такие схемы создаются для каждого сеанса, в котором появляются временные объекты, поэтому их может быть несколько.
Имя схемы для временных объектов текущего сеанса можно получить, обратившись к системной функции:

```sql
SELECT pg_my_temp_schema()::regnamespace;

 pg_my_temp_schema 
-------------------
 pg_temp_4
(1 row)
```

Однако в большинстве случаев точное имя схемы знать не нужно, поскольку при необходимости к временному объекту можно обратиться, используя имя схемы `pg_temp`:

```sql
SELECT * FROM pg_temp.t;

 n 
---
(0 rows)
```


### Функции

Посмотрим на функции командой `\df`.
Модификатор `S` выводит не только пользовательские, но и системные объекты.
С помощью шаблона, который пишется через пробел, можно ограничить выборку:

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

Посмотрим на конкретную функцию командой `\sf`:

```sql
\sf pg_catalog.pg_database_size(oid)

CREATE OR REPLACE FUNCTION pg_catalog.pg_database_size(oid)
 RETURNS bigint
 LANGUAGE internal
 PARALLEL SAFE STRICT
AS $function$pg_database_size_oid$function$
```


## Изучение структуры системного каталога

Чтобы посмотреть, какие запросы на самом деле выполняет `psql`, можно установить переменную `ECHO_HIDDEN`:

```sql
\set ECHO_HIDDEN on
```

```sql
\dt employees

********* QUERY **********
SELECT n.nspname as "Schema",
  c.relname as "Name",
  CASE c.relkind WHEN 'r' THEN 'table' WHEN 'v' THEN 'view' WHEN 'm' THEN 'materialized view' WHEN 'i' THEN 'index' WHEN 'S' THEN 'sequence' WHEN 's' THEN 'special' WHEN 't' THEN 'TOAST table' WHEN 'f' THEN 'foreign table' WHEN 'p' 
THEN 'partitioned table' WHEN 'I' THEN 'partitioned index' END as "Type",
  pg_catalog.pg_get_userbyid(c.relowner) as "Owner"
FROM pg_catalog.pg_class c
     LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
     LEFT JOIN pg_catalog.pg_am am ON am.oid = c.relam
WHERE c.relkind IN ('r','p','t','s','')
  AND c.relname OPERATOR(pg_catalog.~) '^(employees)$' COLLATE pg_catalog.default
  AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 1,2;
**************************

           List of relations
 Schema |   Name    | Type  |  Owner
--------+-----------+-------+----------
 public | employees | table | postgres
(1 row)
```

```sql
\unset ECHO_HIDDEN
```


## OID и reg-типы

Описания таблиц и представлений хранятся в `pg_class`.
Столбцы располагаются в отдельной таблице `pg_attribute`.
Получим список столбцов конкретной таблицы, соединим `pg_class` и `pg_attribute`:

```sql
SELECT a.attname, a.atttypid FROM pg_attribute a WHERE a.attrelid = (
  SELECT oid FROM pg_class WHERE relname = 'employees'
) AND a.attnum > 0;

 attname | atttypid 
---------+----------
 id      |       23
 name    |       25
 manager |       23
(3 rows)
```

Столбцы:
- `attname` - название колонки
- `atttypid` - `OID` или уникальный идентификатор для таблиц системного каталога

`reg-типы` приводят текстовое имя объекта к типу `OID` или наоборот - тип `OID` преобразуют в текстовое имя.
Используя `reg-типы`, запрос можно написать проще, без явного обращения к `pg_class`.
Преобразуем строку "employees" к типу `OID` с помощью конструкции `::regclass`:

```sql
SELECT a.attname, a.atttypid FROM pg_attribute a WHERE a.attrelid = 'employees'::regclass AND a.attnum > 0;

 attname | atttypid 
---------+----------
 id      |       23
 name    |       25
 manager |       23
(3 rows)
```

Аналогично мы можем вывести `OID` как текстовое значение с помощью конструкции `::regtype`:

```sql
SELECT a.attname, a.atttypid::regtype FROM pg_attribute a WHERE a.attrelid = 'employees'::regclass AND a.attnum > 0;

 attname | atttypid 
---------+----------
 id      | integer
 name    | text
 manager | integer
(3 rows)
```

Полный список `reg-типов`:
```sql
\dT reg*

                        List of data types                         
   Schema   |     Name      |             Description              
------------+---------------+--------------------------------------
 pg_catalog | regclass      | registered class                     
 pg_catalog | regcollation  | registered collation
 pg_catalog | regconfig     | registered text search configuration
 pg_catalog | regdictionary | registered text search dictionary
 pg_catalog | regnamespace  | registered namespace
 pg_catalog | regoper       | registered operator
 pg_catalog | regoperator   | registered operator (with args)
 pg_catalog | regproc       | registered procedure
 pg_catalog | regprocedure  | registered procedure (with args)
 pg_catalog | regrole       | registered role
 pg_catalog | regtype       | registered type
(11 rows)
```
