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

