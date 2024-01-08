# Базы данных и схемы

## Базы данных

Инициализация кластера создает `3` базы данных:
```sql
\l

List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(3 rows)
```
Имеющиеся базы данных:
- `postgres` - БД, созданная по умолчанию
- `template0` - шаблон, нужен для работы утилиты `pg_dump`
- `template1` - шаблон, нужен для создания других БД

Также список баз данных можно посмотреть в самой БД:
```sql
SELECT datname, datistemplate, datallowconn, datconnlimit FROM pg_database;

  datname  | datistemplate | datallowconn | datconnlimit 
-----------+---------------+--------------+--------------
 postgres  | f             | t            |           -1
 template1 | t             | t            |           -1
 template0 | t             | f            |           -1
(3 rows)
```

Колонки:
- `datistemplate` - является ли база данных шаблоном
- `datallowconn` - разрешены ли соединения с базой данных
- `datconnlimit` - максимальное количество соединений (`-1` означает "без ограничений")

По умолчанию к БД `template0`  подключения запрещены.


## Создание базы данных из шаблона

Новая БД всегда создается из существующей путем клонирования шаблона (по умолчанию `template1`).
Подключимся к шаблонной БД `template1`:
```sql
\c template1

You are now connected to database "template1" as user "postgres".
```

Вызовем функцию `digest`, вычисляющую хэш-код текстовой строки:
```sql
SELECT digest('Hello, world!', 'md5');

ERROR:  function digest(unknown, unknown) does not exist
LINE 1: SELECT digest('Hello, world!', 'md5');
               ^
HINT:  No function matches the given name and argument types. You might need to add explicit type casts.
```

Такой функции нет...

На самом деле `digest` определена в расширении `pgcrypto`.
Расширение предварительно установлено в ОС.
Чтобы использовать функции, которые входят в его состав, необходимо в БД выполнить команду `CREATE EXTENSION`.
После этого выполняется специальный SQL-скрипт, который создает необходимые объекты в самой БД: функции, представления, и т.д.
```sql
CREATE EXTENSION pgcrypto;

CREATE EXTENSION
```

Теперь функция доступна:
```sql
SELECT digest('Hello, world!', 'md5');

               digest               
------------------------------------
 \x6cd3556deb0da54bca060b4c39479839
(1 row)
```

Например, мы считаем, что расширение `pgcrypto` необходимо в каждой новой БД, которую мы будем создавать.
Чтобы каждый раз не выполнять команду `CREATE EXTENSION` в каждой новой БД, которую мы создаем, мы создали это расширение в БД `template1`.

Создадим БД:
```sql
CREATE DATABASE db;

CREATE DATABASE
```

```sql
SELECT datname, datistemplate, datallowconn, datconnlimit FROM pg_database;

  datname  | datistemplate | datallowconn | datconnlimit 
-----------+---------------+--------------+--------------
 postgres  | f             | t            |           -1
 template1 | t             | t            |           -1
 template0 | t             | f            |           -1
 db        | f             | t            |           -1
(4 rows)
```

Подключимся к ней:
```bash
\c db

You are now connected to database "db" as user "postgres".
```

Поскольку для создания БД по умолчанию используется шаблон `template1`, в новой БД также доступны функции пакета `pgcrypto`:
```sql
SELECT digest('Hello, world!', 'md5');

               digest               
------------------------------------
 \x6cd3556deb0da54bca060b4c39479839
(1 row)
```


## Управление базами данных

Созданную БД можно переименовывать (к ней не должно быть подключений):
```bash
\c postgres

You are now connected to database "postgres" as user "postgres".
```

```sql
ALTER DATABASE db RENAME TO appdb;

ALTER DATABASE
```

```sql
SELECT datname, datistemplate, datallowconn, datconnlimit FROM pg_database;

  datname  | datistemplate | datallowconn | datconnlimit 
-----------+---------------+--------------+--------------
 postgres  | f             | t            |           -1
 template1 | t             | t            |           -1
 template0 | t             | f            |           -1
 appdb     | f             | t            |           -1
(4 rows)
```

Можно изменять и другие параметры, например максимальное количество одновременных подключений:
```sql
ALTER DATABASE appdb CONNECTION LIMIT 10;

ALTER DATABASE
```

```sql
SELECT datname, datistemplate, datallowconn, datconnlimit FROM pg_database;

  datname  | datistemplate | datallowconn | datconnlimit 
-----------+---------------+--------------+--------------
 postgres  | f             | t            |           -1
 template1 | t             | t            |           -1
 template0 | t             | f            |           -1
 appdb     | f             | t            |           10
(4 rows)
```

Размер БД можно узнать с помощью функции:
```sql
SELECT pg_database_size('appdb');

 pg_database_size 
------------------
          8782627
(1 row)
```

Чтобы не считать разряды, можно вывести размер в читаемом виде:
```sql
SELECT pg_size_pretty(pg_database_size('appdb'));

 pg_size_pretty 
----------------
 8577 kB
(1 row)
```


## Схемы

Объекты мы всегда располагаем в каких-то схемах.

Список схем можно узнать командой `dn` (`describe namespace`):
```sql
\dn

  List of schemas
  Name  |  Owner
--------+----------
 public | postgres
(1 row)
```
По умолчанию команда не показывает системные схемы.

Создадим новую схему:
```sql
\c appdb

You are now connected to database "appdb" as user "postgres".
```

```sql
CREATE SCHEMA app;

CREATE SCHEMA
```

```sql
\dn

  List of schemas
  Name  |  Owner
--------+----------
 app    | postgres
 public | postgres
(2 rows)
```

Если теперь создать таблицу и не указать имя схемы, в какую схему она попадет?

Надо посмотреть на путь поиска.

```sql
SHOW search_path;

   search_path   
-----------------
 "$user", public
(1 row)
```

Конструкция "$user" обозначает схему с тем же именем, что и имя текущего пользователя (в нашем случае "postgres").
Поскольку такой схемы нет, она игнорируется.

Чтобы не думать над тем, какие схемы есть, каких нет, и какие не указаны явно, можно воспользоваться функцией:

```sql
SELECT current_schemas(true);

   current_schemas   
---------------------
 {pg_catalog,public}
(1 row)
```

Теперь создадим таблицу:
```sql
CREATE table t(s text);

CREATE TABLE
```

```sql
INSERT INTO t VALUES ('Я - таблица t');

INSERT 0 1
```

Список таблиц можно получить командой `\dt`:

```sql
\dt

        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t    | table | postgres
(1 row)
```















---

Схемы

Пространство имен для объектов внутри базы данных.

Каждый объект принадлежит какой-либо схеме.


Задачи

Разделение объектов на логические группы.

Предотвращение конфликта имен между приложениями.

Схема и пользователь - разные сущности.


Схема `pg_catalog` - системный каталог, который хранит информацию о том, что находится внутри базы данных.

В каждой базе данных по умолчанию есть схема `public`.


---



Путь поиска

Определение схемы объекта

Квалифицированное имя (схема.имя) явно определяет схему

Имя без квалификатора проверяется в схемах, указанных в пути поиска


Путь поиска

Задается параметром `search_path`

Исключаются несуществующие схемы и схемы, к которым нет доступа.

Подставляются неявно подразумеваемые схемы.

Реальное значение показывает функция `current_schemas`.

Первая явно указанная в пути схема используется для создания объектов.


---


Специальные схемы

Схема public

По умолчанию входит в путь поиска.

Если ничего не менять, все объекты будут в этой схеме.


Схема, совпадающая по имени с пользователем.

По умолчанию входит в путь поиска, но не существует.

Если создать, объекты пользователя будут в этой схеме.


Схема pg_catalog

Схема для объектов системного каталога.

Если этой схемы нет в пути, она неявно подразумевается первой.


Временные таблицы

Существуют на время сеанса или транзакции.

Не журналируются (невозможно восстановление после сбоя).

Не попадают в общий буферный кэш.


Схема `pg_temp_<number>`

Автоматически создается для временных таблиц.

pg_temp - ссылка на конкретную временную схему данного сеанса.

Если pg_temp нет в пути, она неявно подразумевается самой первой.

По окончании сеанса все объекты временной схемы удаляются, а сама схема остается и повторно используется для других сеансов.





















