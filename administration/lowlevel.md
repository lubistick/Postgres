# Низкий уровень

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




