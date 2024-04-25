# Табличные пространства

Табличное пространство (ТП) - каталог, в котором хранятся файлы.
При создании кластера создаются два ТП:

```sql
SELECT * FROM pg_tablespace;

 oid  |  spcname   | spcowner | spcacl | spcoptions 
------+------------+----------+--------+------------
 1663 | pg_default |       10 |        |
 1664 | pg_global  |       10 |        |
(2 rows)
```

- `pg_global` - табличное пространство для общих объектов кластера, располагается в каталоге `$PGDATA/global/`
- `pg_default` - табличное пространство по умолчанию, располагается в каталоге `$PGDATA/base/<oid_БД>/`
- пользовательские табличные пространства располагаются в каталоге `$PGDATA/pg_tblspc/<oid_табличного_пространства>/<символическая_ссылка>`


## Создание табличного пространства

Для нового ТП нужен пустой каталог, владельцем которого является пользователь `postgres`.

```bash
mkdir /var/lib/postgresql/ts_dir
```

```bash
chown postgres /var/lib/postgresql/ts_dir
```

Создадим ТП:

```sql
CREATE TABLESPACE ts LOCATION '/var/lib/postgresql/ts_dir';

CREATE TABLESPACE
```

Получим список ТП командой `psql`:

```sql
\db

                List of tablespaces
    Name    |  Owner   |          Location
------------+----------+----------------------------
 pg_default | postgres |
 pg_global  | postgres |
 ts         | postgres | /var/lib/postgresql/ts_dir
(3 rows)
```

Получим `oid` созданного ТП:
```sql
SELECT oid FROM pg_tablespace WHERE spcname = 'ts';

  oid  
-------
 82125
(1 row)
```

Посмотрим символическую ссылку:

```bash
ls -l /var/lib/postgresql/data/pg_tblspc/

total 0
lrwxrwxrwx    1 postgres postgres        26 Apr 23 15:06 82125 -> /var/lib/postgresql/ts_dir
```


## Табличные пространства для баз данных

У каждой БД есть ТП по умолчанию.
Создадим БД `appdb` и назначим ей ТП `ts` по умолчанию:
```sql
CREATE DATABASE appdb TABLESPACE ts;

CREATE DATABASE
```
Теперь все созданные таблицы и индексы будут попадать в ТП `ts`, если явно не указать другое.

Подключимся к БД:
```sql
\c appdb

You are now connected to database "appdb" as user "postgres".
```

Создадим таблицу:
```sql
CREATE TABLE t1(id serial, name text);

CREATE TABLE
```

Создадим еще одну таблицу в другом ТП `pg_default`:
```sql
CREATE TABLE t2(id serial, name text) TABLESPACE pg_default;

CREATE TABLE
```

Посмотрим ТП для созданных выше таблиц:
```sql
SELECT tablename, tablespace FROM pg_tables WHERE schemaname = 'public';

 tablename | tablespace 
-----------+------------
 t1        |
 t2        | pg_default
(2 rows)
```
- у таблицы `t1` поле `tablespace` пустое, т.е. имеет значение `ts` - табличное пространство по умолчанию для текущей БД.
- у таблицы `t2` поле `tablespace` имеет значение `pg_default`.


Одно табличное пространство может использоваться для объектов нескольких БД.
Создадим БД `configdb` с ТП `pg_default`:

```sql
CREATE DATABASE configdb;

CREATE DATABASE
```

У этой БД табличным пространством по умолчанию будет `pg_default`.

```sql
\c configdb

You are now connected to database "configdb" as user "postgres".
```

Создадим таблицу в ТП `ts`:
```sql
CREATE TABLE t(n integer) TABLESPACE ts;

CREATE TABLE
```


## Перемещение объектов в другие табличные пространства

Таблицы и индексы можно перемещать между ТП.
Это физическая операция - файлы копируются из одного каталога в другой.
Таблица блокируется, пока выполняется операция.

```sql
\c appdb

You are now connected to database "appdb" as user "postgres".
```


### Перемещение таблицы

Переместим таблицу `t1` в из ТП `ts` в `pg_default`:
```sql
ALTER TABLE t1 SET TABLESPACE pg_default;

ALTER TABLE
```

```sql
SELECT tablename, tablespace FROM pg_tables WHERE schemaname = 'public';

 tablename | tablespace 
-----------+------------
 t2        | pg_default
 t1        | pg_default
(2 rows)
```


### Перемещение всех объектов табличного пространства

Переместим все объекты из ТП `pg_default` в `ts`:
```sql
ALTER TABLE ALL IN TABLESPACE pg_default SET TABLESPACE ts;

ALTER TABLE
```

```sql
SELECT tablename, tablespace FROM pg_tables WHERE schemaname = 'public';

 tablename | tablespace 
-----------+------------
 t2        |
 t1        |
(2 rows)
```


## Размер табличного пространства

Посмотрим объем ТП `ts`:
```sql
SELECT pg_size_pretty(pg_tablespace_size('ts'));

 pg_size_pretty 
----------------
 8689 kB
(1 row)
```

Почему такой размер ТП, если в нем всего несколько пустых таблиц?
Поскольку `ts` является ТП по умолчанию для базы `appdb`, в нем хранятся объекты системного каталога.
Они и занимают место.


## Удаление табличного пространства

Попробуем удалить ТП:

```sql
DROP TABLESPACE ts;

ERROR:  tablespace "ts" is not empty
```

ТП можно удалить, только если оно пустое.
В команде `DROP TABLESPACE` нельзя использовать фразу `CASCADE`, т.к. объекты ТП могут принадлежать разным БД, а подключены мы только к одной.
Выясним, в каких БД есть зависимые объекты.

Узнаем `OID` ТП:
```sql
SELECT oid FROM pg_tablespace WHERE spcname = 'ts';

  oid  
-------
 73917
(1 row)
```

Запомним `OID` в переменную `psql`:
```sql
SELECT oid AS tsoid FROM pg_tablespace WHERE spcname = 'ts' \gset
```

Посмотрим список БД, в которых есть объекты из удаляемого ТП:
```sql
SELECT datname FROM pg_database WHERE oid IN (SELECT pg_tablespace_databases(:tsoid));

 datname  
----------
 appdb
 configdb
(2 rows)
```

Подключимся сначала к БД `configdb`:
```sql
\c configdb

You are now connected to database "configdb" as user "postgres".
```

Получим список объектов из `pg_class`:
```sql
SELECT relnamespace::regnamespace, relname, relkind FROM pg_class WHERE reltablespace = :tsoid;

 relnamespace | relname | relkind 
--------------+---------+---------
 public       | t       | r
(1 row)
```

Тут только одна таблица `t`. Удалим ее:
```sql
DROP TABLE t;

DROP TABLE
```

Теперь подключимся к БД `appdb`:
```sql
\c appdb

You are now connected to database "appdb" as user "postgres".
```

ТП `ts` выбрано по умолчанию в БД `appdb`, поэтому ищем объекты в `pg_class`, где поле `reltablespace` содержит `0`.
Это объекты системного каталога:
```sql
SELECT count(*) FROM pg_class WHERE reltablespace = 0;

 count 
-------
   361
(1 row)
```

Отключимся от БД, чтобы сменить ТП по умолчанию:
```sql
\c postgres

You are now connected to database "postgres" as user "postgres".
```

Меняем. При этом все таблицы из старого ТП физически переносятся в новое:
```sql
ALTER DATABASE appdb SET TABLESPACE pg_default;

ALTER DATABASE
```

```sql
\c appdb

You are now connected to database "appdb" as user "postgres".
```

Теперь удалим ТП:
```sql
DROP TABLESPACE ts;

DROP TABLESPACE
```


## Табличное пространство у базы данных `template1`

Создадим ТП:
```sql
CREATE TABLESPACE ts LOCATION '/var/lib/postgresql/ts_dir';

CREATE TABLESPACE
```

Поменяем ТП по умолчанию для БД `template1`:
```sql
ALTER DATABASE template1 SET TABLESPACE ts;

ALTER DATABASE
```

Создадим новую БД `db`:
```sql
CREATE DATABASE db;

CREATE DATABASE
```

Табличное пространство по умолчанию определяется шаблоном, из которого клонируется новая БД:
```sql
SELECT spcname FROM pg_tablespace WHERE oid = (SELECT dattablespace FROM pg_database WHERE datname = 'db');

 spcname 
---------
 ts
(1 row)
```

Сменим ТП на `pg_default`:
```sql
ALTER DATABASE template1 SET TABLESPACE pg_default;

ALTER DATABASE
```

Удалим БД:
```sql
DROP DATABASE db;

DROP DATABASE
```

Удалим ТП:
```sql
DROP TABLESPACE ts;

DROP TABLESPACE
```


## Установка `random_page_cost` для табличного пространства.

Параметры `seq_page_cost` и `random_page_cost` используются планировщиком запросов
и задают примерную стоимость чтения с диска одной страницы данных при [последовательном](../optimization/seqscan.md) и произвольном доступе соответственно. 
Чем меньше соотношение между этими параметрами, тем чаще планировщик будет предпочитать индексный доступ последовательному сканированию таблицы.

Значения по умолчанию параметров `seq_page_cost` и `random_page_cost` больше подходят для медленных HDD-дисков.
Предполагается, что доступ к произвольной странице данных в `4` раза дороже последовательного:
```sql
SELECT name, setting FROM pg_settings WHERE name IN ('seq_page_cost', 'random_page_cost');

       name       | setting 
------------------+---------
 random_page_cost | 4
 seq_page_cost    | 1
(2 rows)
```

Если используются диски с разными характеристиками, для них можно создать разные табличные пространства и настроить подходящие соотношения этих параметров.
Для быстрых SSD-дисков значение `random_page_cost` можно уменьшить практически до значения `seq_page_cost`:

```sql
ALTER TABLESPACE pg_default SET (random_page_cost = 1.1);

ALTER TABLESPACE
```

Настройки, сделанные командой `ALTER TABLESPACE`, сохраняются в таблице `pg_tablespace`.
Посмотрим их в `psql` командой `\db+`:
```sql
\db+

                                          List of tablespaces
    Name    |  Owner   | Location | Access privileges |        Options         |  Size   | Description
------------+----------+----------+-------------------+------------------------+---------+-------------
 pg_default | postgres |          |                   | {random_page_cost=1.1} | 2942 MB |
 pg_global  | postgres |          |                   |                        | 576 kB  |
(2 rows)
```

Параметры `seq_page_cost` и `random_page_cost` также можно установить в `postgresql.conf`, тогда они будут действовать для всех табличных пространств.
