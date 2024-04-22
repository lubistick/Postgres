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

