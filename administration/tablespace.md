# Табличные пространства

Табличное пространство - каталог, в котором хранятся файлы.

Таб пр pg_global -> $PGDATA/global/
Таб пр pg_default -> $PGDATA/base/dboid/
Таб пр -> $PGDATA/pg_tblspc/tsoid/<символическая_ссылка>


## Служебные табличные пространства

При создании кластера создаются два табличных пространства:

```sql
SELECT * FROM pg_tablespace;

 oid  |  spcname   | spcowner | spcacl | spcoptions 
------+------------+----------+--------+------------
 1663 | pg_default |       10 |        |
 1664 | pg_global  |       10 |        |
(2 rows)
```

- `pg_global` - общие объекты кластера
- `pg_default` - табличное пространство по умолчанию


## Пользовательские табличные пространства

Для нового табличного пространства нужен пустой каталог, владельцем которого является пользователь postgres.

```bash
mkdir /var/lib/postgresql/ts_dir
```

```bash
chown postgres /var/lib/postgresql/ts_dir
```

Теперь можно создать табличное пространство:

```sql
CREATE TABLESPACE ts LOCATION '/var/lib/postgresql/ts_dir';

CREATE TABLESPACE
```

Список табличных пространств можно получить и командой psql:

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

Для стандартных табличных пространств расположение не указывается, потому что оно и так понятно.


У каждой БД есть табличное пространство "по умолчанию".
Создадим БД и назначим ей ts в качестве такого пространства:
```sql
CREATE DATABASE appdb TABLESPACE ts;

CREATE DATABASE
```

Теперь все созданные таблицы и индексы будут попадать в ts, если явно не указать другое.

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

И создадим еще одну таблицу в другом табличном пространстве:
```sql
CREATE TABLE t2(id serial, name text) TABLESPACE pg_default;

CREATE TABLE
```

```sql
SELECT tablename, tablespace FROM pg_tables WHERE schemaname = 'public';

 tablename | tablespace 
-----------+------------
 t1        |
 t2        | pg_default
(2 rows)
```

Пустое поле tablespace указывает на табличное пространство по умолчанию для текущей БД, а у второй таблицы поле заполнено.

Одно табличное пространство может использоваться для объектов нескольких БД.

```sql
CREATE DATABASE configdb;

CREATE DATABASE
```

У этой БД табличным пространством по умолчанию будет `pg_default`.

```sql
\c configdb

You are now connected to database "configdb" as user "postgres".
```

```sql
CREATE TABLE t(n integer) TABLESPACE ts;

CREATE TABLE
```


## Управление объектами в табличных пространствах

Таблицы и индексы можно перемещать между табличными пространствами.
Это физическая операция - файлы копируются из одного каталога в другой.
Таблица блокируется, пока выполняется операция.

```sql
\c appdb

You are now connected to database "appdb" as user "postgres".
```

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


Можно переместить и все объекты из одного табличного пространства в другое:
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

Мы уже рассматривали, как узнать объем, занимаемый базой данных.
Объем можно посмотреть и в разрезе табличных пространств:
```sql
SELECT pg_size_pretty(pg_tablespace_size('ts'));

 pg_size_pretty 
----------------
 8689 kB
(1 row)
```

Почему такой размер, если в табличном пространстве всего несколько пустых таблиц?
Поскольку `ts` является табличным пространством по умолчанию для базы `appdb`, в нем хранятся объекты системного каталога.
Они и занимают место.


## Удаление табличного пространства

Табличное пространство можно удалить, но только в том случае, если оно пустое.

```sql
DROP TABLESPACE ts;

ERROR:  tablespace "ts" is not empty
```

В отличие от удаления схемы, в команде `DROP TABLESPACE` нельзя использовать фразу `CASCADE`:
объекты табличного пространства могут принадлежать разным БД, а подключены мы только к одной.

Но можно выяснить, в каких БД есть зависимые объекты.
В этом нам поможет системный каталог.
Сначала узнаем и запомним `OID` табличного пространства:

```sql
SELECT oid FROM pg_tablespace WHERE spcname = 'ts';

  oid  
-------
 73917
(1 row)
```

```sql
SELECT oid AS tsoid FROM pg_tablespace WHERE spcname = 'ts' \gset
```

Затем получим список БД, в которых есть объекты из удаляемого пространства:
```sql
SELECT datname FROM pg_database WHERE oid IN (SELECT pg_tablespace_databases(:tsoid));

 datname  
----------
 appdb
 configdb
(2 rows)
```

Дальше подключаемся к каждой БД и получаем список объектов из `pg_class`:
```sql
\c configdb

You are now connected to database "configdb" as user "postgres".
```

```sql
SELECT relnamespace::regnamespace, relname, relkind FROM pg_class WHERE reltablespace = :tsoid;

 relnamespace | relname | relkind 
--------------+---------+---------
 public       | t       | r
(1 row)
```

Таблица `t` больше не нужна, удалим ее:
```sql
DROP TABLE t;

DROP TABLE
```

И вторая БД.
Но поскольку `ts` является табличным пространством по умолчанию, у объектов в `pg_class` поле содержит ноль:

```sql
\c appdb

You are now connected to database "appdb" as user "postgres".
```

```sql
SELECT count(*) FROM pg_class WHERE reltablespace = 0;

 count 
-------
   361
(1 row)
```

Это как нам уже известно, объекты системного каталога.

Табличное пространство по умолчанию можно сменить.
При этом все таблицы из старого пространства физически переносятся в новое.
Предварительно надо отключиться от БД:

```sql
\c postgres

You are now connected to database "postgres" as user "postgres".
```

