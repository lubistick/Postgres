# Привилегии

Привилегии - это право выполнять действия с объектами кластера `Postgres`.
Привилегии существуют для:
- баз данных
- схем
- временных схем
- табличных пространств
- таблиц
- представлений
- последовательностей
- функций

Категории ролей с точки зрения управления доступом:
- `суперпользователь` - имеет полный доступ ко всем объектам (проверки доступа не выполняются)
- `владелец объекта` - изначально получает полный набор привилегий, а также право на удаление объекта, выдачу и отзыв привилегий на объект
- `остальные роли` - доступ в рамках выданных привилегий


## Скрытая роль public

Роль `public` не отображается в списке ролей, но она существует.
Все другие роли неявно входят в групповую роль `public`, а значит, получают ее привилегии.
По умолчанию роль `public` имеет следующие привилегии:
- для любой БД:
  - `CONNECT` - право подключаться к БД (не путаем с атрибутом `LOGIN` у роли, который позволяет подключаться к кластеру `Postgres`)
  - `TEMPORARY` право на создание временных таблиц внутри БД (привилегия `TEMPORARY` выдается для БД, а не для схемы, т.к. точное имя схемы для временных объектов заранее неизвестно)
- для схемы `public`:
  - `CREATE` - право создавать объекты внутри схемы
  - `USAGE` - право доступа к объектам внутри схемы
- для схем `pg_catalog` и `information_schema`:
  - `USAGE` - право доступа к объектам внутри схемы
- для любой функции:
  - `EXECUTE` право выполнять функцию


## Базы данных

Для БД существуют следующие привилегии:
- `CONNECT` - разрешает подключение к этой БД 
- `CREATE` разрешает создавать схемы в этой БД
- `TEMPORARY` - разрешает создание временных таблиц 

Создадим БД `access_privileges`:
```sql
CREATE DATABASE access_privileges;

CREATE DATABASE
```

Подключимся к ней:
```sql
\c access_privileges

You are now connected to database "access_privileges" as user "postgres".
```

Посмотрим информацию о БД.
В колонке `Access privileges` хранятся выданные привилегии.
Пустое поле `Access privileges` означает,
что у владельца (колонка `Owner`) есть полный набор привилегий, а у всех остальных привилегий нет:
```sql
\l access_privileges

                                   List of databases
       Name        |  Owner   | Encoding |  Collate   |   Ctype    | Access privileges
-------------------+----------+----------+------------+------------+-------------------
 access_privileges | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
(1 row)
```

Создадим пользователя `application`:
```sql
CREATE ROLE application LOGIN;

CREATE ROLE
```

Пользователь `application` не имеет право подключаться к БД - не имеет привилегию `CONNECT`.
Но он все равно может это сделать, т.к. любой пользователь входит в групповую роль `public`, у которой эта привилегия есть:
```sql
\c - application

You are now connected to database "access_privileges" as user "application".
```

А вот создать схему внутри БД не может, т.к. не имеет привилегию `CREATE` для БД `access_privileges`:
```sql
CREATE SCHEMA application;

ERROR:  permission denied for database access_privileges
```


### Выдача привилегий

Право выдачи и отзыва привилегий на объект имеют:
- суперпользователь
- владелец объекта

Подключимся под суперпользователем:
```sql
\c - postgres

You are now connected to database "access_privileges" as user "postgres".
```

Дадим пользователю `application` привилегию на создание схем (`CREATE`) внутри БД `access_privileges`:
```sql
GRANT CREATE ON DATABASE access_privileges TO application;

GRANT
```

Посмотрим информацию о БД `access_privileges`:
```sql
\l access_privileges

                                     List of databases
       Name        |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-------------------+----------+----------+------------+------------+------------------------
 access_privileges | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =Tc/postgres          +
                   |          |          |            |            | postgres=CTc/postgres +
                   |          |          |            |            | application=C/postgres
(1 row)
```
Теперь колонка `Access privileges` содержит в себе строку, разделенную на подстроки.
В каждой подстроке записана привилегия на конкретного пользователя в формате: `<роль>=<привилегии>/<кем_выданы>`.
Названия привилегий обозначаются одной буквой:
- `C` - `CREATE`
- `c` - `CONNECT`
- `T` - `TEMPORARY`

Смотрим колонку `Access privileges`:
- первая подстрока - пустота слева от знака равенства обозначает роль `public`, у которой есть привилегии создавать временные схемы (`T`) и подключаться к БД (`c`)
- третья подстрока - пользователь `application` получил привилегию на создание схем (`C`) от суперпользователя `postgres`.

Подключимся под пользователем `application`:
```sql
\c - application

You are now connected to database "access_privileges" as user "application".
```

Теперь пользователь может создать схему:
```sql
CREATE SCHEMA application;

CREATE SCHEMA
```


### Отзыв привилегий

Подключимся под суперпользователем:
```sql
\c - postgres

You are now connected to database "access_privileges" as user "postgres".
```

Отберем привилегию `CREATE` для БД `access_privileges` у пользователя `application`:
```sql
REVOKE CREATE ON DATABASE access_privileges FROM application;

REVOKE
```

Проверим:
```sql
\l access_privileges
                                     List of databases
       Name        |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-------------------+----------+----------+------------+------------+-----------------------
 access_privileges | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =Tc/postgres         +
                   |          |          |            |            | postgres=CTc/postgres
(1 row)
```


## Схемы

Для схем существуют следующие привилегии:
- `CREATE (C)` - разрешает создавать объекты в этой схеме
- `USAGE (U)` - разрешает обращаться к объектам этой схемы

Подключимся под пользователем `application`:
```sql
\c - application

You are now connected to database "access_privileges" as user "application".
```

Пользователь `application` является владельцем схемы `application`, потому что он ее создал:
```sql
\dn

      List of schemas
    Name     |    Owner
-------------+-------------
 application | application
 public      | postgres
(2 rows)
```

Создадим таблицу `t1` с одной колонкой.
Она будет создана в схеме `application`, т.к. по умолчанию схема равна имени пользователя:
```sql
CREATE TABLE t1(n integer);

CREATE TABLE
```

Подключимся под суперпользователем:
```sql
\c - postgres

You are now connected to database "access_privileges" as user "postgres".
```

Создадим пользователя `microservice`:
```sql
CREATE ROLE microservice LOGIN;

CREATE ROLE
```

Подключимся под пользователем `microservice`:
```sql
\c - microservice

You are now connected to database "access_privileges" as user "microservice".
```

У пользователя `microservice` нет прав обращаться к объектам схемы `application`.
Попробуем прочитать таблицу `t1`:
```sql
SELECT * FROM application.t1;

ERROR:  permission denied for schema application
LINE 1: SELECT * FROM application.t1;
```

Подключимся под пользователем `application`:
```sql
\c - application

You are now connected to database "access_privileges" as user "application".
```

Выдадим все привилегии на схему `application` пользователю `microservice`, с помощью ключевого слова `ALL`:
```sql
GRANT ALL ON SCHEMA application TO microservice;

GRANT
```

Проверим привилегии, смотрим вторую подстроку колонки `Access privileges`:
```sql
\dn+ application

                            List of schemas
    Name     |    Owner    |      Access privileges      | Description
-------------+-------------+-----------------------------+-------------
 application | application | application=UC/application +|
             |             | microservice=UC/application |
(1 row)
```


## Таблицы

Больше всего привилегий определено для таблиц:
- `INSERT (a)` - право на вставку строк (append)
- `SELECT (r)` - право читать таблицу (read)
- `UPDATE (w)` - право обновлять строки (write)
- `DELETE (d)` - право удалять строки из таблицы (delete)
- `TRUNCATE (D)` - право на очистку таблицы
- `REFERENCES (x)` - право изменять внешние ключи (foreign keys)
- `TRIGGER (t)` - право создавать триггеры

Подключимся под пользователем `microservice`:
```sql
\c - microservice

You are now connected to database "access_privileges" as user "microservice".
```

У пользователя `microservice` есть привилегии обращаться к схеме `application`,
но нет привилегий обращаться к таблице `t1`:
```sql
SELECT * FROM application.t1;

ERROR:  permission denied for table t1
```

Подключимся под пользователем `application`:
```sql
\c - application

You are now connected to database "access_privileges" as user "application".
```

Дадим пользователю `microservice` права на чтение таблицы `t1` (`SELECT`):
```sql
GRANT SELECT ON t1 TO microservice;

GRANT
```

Посмотрим привилегии на таблицу `t1`:
```sql
\dp t1

                                      Access privileges
   Schema    | Name | Type  |        Access privileges        | Column privileges | Policies
-------------+------+-------+---------------------------------+-------------------+----------
 application | t1   | table | application=arwdDxt/application+|                   |
             |      |       | microservice=r/application      |                   |
(1 row)
```

Подключимся под пользователем `microservice`:
```sql
\c - microservice

You are now connected to database "access_privileges" as user "microservice".
```

Теперь `microservice` может прочитать таблицу `t1`:
```sql
SELECT * FROM application.t1;

 n 
---
(0 rows)
```

Но вставить строку не сможет:
```sql
INSERT INTO application.t1 VALUES (42);

ERROR:  permission denied for table t1
```


### Удаление объекта

Привилегий на удаление таблицы не существует:
```sql
DROP TABLE application.t1;

ERROR:  must be owner of table t1
```

Удалить таблицу может только владелец или суперпользователь.
Подключимся под пользователем `application`:
```sql
\c - application

You are now connected to database "access_privileges" as user "application".
```

Владелец может удалить таблицу:
```sql
DROP TABLE t1;

DROP TABLE
```


## Колонки таблицы

Некоторые привилегии можно выдать на определенные колонки.
Создадим таблицу `t2` с двумя колонками:
```sql
CREATE TABLE t2(n integer, m integer);

CREATE TABLE
```

Дадим пользователю `microservice` право на вставку в обе колонки:
```sql
GRANT INSERT(n, m) ON t2 TO microservice;

GRANT
```

И право читать только колонку `n`:
```sql
GRANT SELECT(n) ON t2 TO microservice;

GRANT
```

Посмотрим привилегии таблицы `t2` - в столбце `Column privileges` появились соответствующие права доступа:
```sql
\dp t2

                                     Access privileges
   Schema    | Name | Type  | Access privileges |       Column privileges       | Policies
-------------+------+-------+-------------------+-------------------------------+----------
 application | t2   | table |                   | n:                           +|
             |      |       |                   |   microservice=ar/application+|
             |      |       |                   | m:                           +|
             |      |       |                   |   microservice=a/application  |
(1 row)
```

Подключимся под пользователем `microservice`:
```sql
\c - microservice

You are now connected to database "access_privileges" as user "microservice".
```

Добавим строку:
```sql
INSERT INTO application.t2 VALUES (1, 2);

INSERT 0 1
```

Все колонки читать не можем:
```sql
SELECT * FROM application.t2;

ERROR:  permission denied for table t2
```

Только колонку `n`:
```sql
SELECT n FROM application.t2;

 n 
---
 1
(1 row)
```


### Изменение строк в таблице

Подключимся под пользователем `application`:
```sql
\c - application

You are now connected to database "access_privileges" as user "application".
```

Дадим пользователю `microservice` право изменять строки (`UPDATE`) в таблице `t2`:
```sql
GRANT UPDATE ON t2 TO microservice;

GRANT
```

Посмотрим привилегии таблицы `t2`:
```sql
\dp t2

                                            Access privileges
   Schema    | Name | Type  |        Access privileges        |       Column privileges       | Policies
-------------+------+-------+---------------------------------+-------------------------------+----------
 application | t2   | table | application=arwdDxt/application+| n:                           +|
             |      |       | microservice=w/application      |   microservice=ar/application+|
             |      |       |                                 | m:                           +|
             |      |       |                                 |   microservice=a/application  |
(1 row)
```

Подключимся под пользователем `microservice`:
```sql
\c - microservice

You are now connected to database "access_privileges" as user "microservice".
```

Интересно то, что мы получим ошибку при изменении строки:
```sql
UPDATE application.t2 SET m = m + 1;

ERROR:  permission denied for table t2
```

Дело в том, что перед изменением строк, идет их чтение.
Но привилегии на чтение столбца `m` нет.
Подключимся под пользователем `application`:
```sql
\c - application

You are now connected to database "access_privileges" as user "application".
```

Дадим право на чтение:
```sql
GRANT SELECT ON t2 TO microservice;

GRANT
```

Подключимся под пользователем `microservice`:
```sql
\c - microservice

You are now connected to database "access_privileges" as user "microservice".
```

Теперь изменение строк работает:
```sql
UPDATE application.t2 SET m = m + 1;

UPDATE 1
```











## Передача права

Создадим третью роль.
Роль `bob` пробует передать ей права на таблицу `t1`, принадлежащую роли `alice`:
```sql
\c - postgres

You are now connected to database "postgres" as user "postgres".
```

```sql
CREATE ROLE charlie LOGIN;

CREATE ROLE
```

У пользователя `bob` есть полный доступ к `t1`:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

```sql
\dp alice.t1

                             Access privileges
 Schema | Name | Type  |  Access privileges  | Column privileges | Policies
--------+------+-------+---------------------+-------------------+----------
 alice  | t1   | table | alice=arwdDxt/alice+|                   |
        |      |       | bob=arwdDxt/alice   |                   |
(1 row)
```

Но он не может передать свои привилегии пользователю `charlie`:
```sql
GRANT SELECT ON alice.t1 TO charlie;

WARNING:  no privileges were granted for "t1"
GRANT
```

Чтобы это было возможно, роль `alice` должна разрешить пользователю `bob` передавать права:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

```sql
GRANT SELECT, UPDATE ON t1 TO bob WITH GRANT OPTION;

GRANT
```

```sql
\dp t1

                             Access privileges
 Schema | Name | Type  |  Access privileges  | Column privileges | Policies
--------+------+-------+---------------------+-------------------+----------
 alice  | t1   | table | alice=arwdDxt/alice+|                   |
        |      |       | bob=ar*w*dDxt/alice |                   |
(1 row)
```

Звездочки справа от символа привилегии показывают право передачи.

Теперь пользователь `bob` может поделиться с пользователем `charlie` привилегиями, в том числе и правом передачи:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

```sql
GRANT SELECT ON alice.t1 TO charlie WITH GRANT OPTION;

GRANT
```

```sql
GRANT UPDATE ON alice.t1 TO charlie;

GRANT
```

```sql
\dp alice.t1

                             Access privileges
 Schema | Name | Type  |  Access privileges  | Column privileges | Policies
--------+------+-------+---------------------+-------------------+----------
 alice  | t1   | table | alice=arwdDxt/alice+|                   |
        |      |       | bob=ar*w*dDxt/alice+|                   |
        |      |       | charlie=r*w/bob     |                   |
(1 row)
```

Роль может получить одну и ту же привилегию от разных ролей.
Обратите внимание, если привилегия выдается суперпользователем, она выдается от имени владельца:
```sql
\c - postgres

You are now connected to database "postgres" as user "postgres".
```

```sql
GRANT UPDATE ON alice.t1 TO charlie;

GRANT
```

```sql
\dp alice.t1

                             Access privileges
 Schema | Name | Type  |  Access privileges  | Column privileges | Policies
--------+------+-------+---------------------+-------------------+----------
 alice  | t1   | table | alice=arwdDxt/alice+|                   |
        |      |       | bob=ar*w*dDxt/alice+|                   |
        |      |       | charlie=r*w/bob    +|                   |
        |      |       | charlie=w/alice     |                   |
(1 row)
```

Роль может отозвать привилегии только у той роли, которой она их непосредственно выдала.
Например, роль `alice` не сможет отозвать право передачи у пользователя `charlie`, потому что она его не выдавала.
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

```sql
REVOKE GRANT OPTION FOR SELECT ON alice.t1 FROM charlie;

REVOKE
```

Никакой ошибки не фиксируется, но и права не изменяются:
```sql
\dp t1

                             Access privileges
 Schema | Name | Type  |  Access privileges  | Column privileges | Policies
--------+------+-------+---------------------+-------------------+----------
 alice  | t1   | table | alice=arwdDxt/alice+|                   |
        |      |       | bob=ar*w*dDxt/alice+|                   |
        |      |       | charlie=r*w/bob    +|                   |
        |      |       | charlie=w/alice     |                   |
(1 row)
```

В то же время роль `alice` не может просто так отозвать привилегии у пользователя `bob`, если он успел передать их кому-либо еще:
```sql
REVOKE GRANT OPTION FOR SELECT ON alice.t1 FROM bob;

ERROR:  dependent privileges exist
HINT:  Use CASCADE to revoke them too.
```

В таком случае привилегии надо отзывать по всей иерархии передачи с помощью `CASCADE`:
```sql
REVOKE GRANT OPTION FOR SELECT ON alice.t1 FROM bob CASCADE;

REVOKE
```

```sql
\dp t1

                             Access privileges
 Schema | Name | Type  |  Access privileges  | Column privileges | Policies
--------+------+-------+---------------------+-------------------+----------
 alice  | t1   | table | alice=arwdDxt/alice+|                   |
        |      |       | bob=arw*dDxt/alice +|                   |
        |      |       | charlie=w/bob      +|                   |
        |      |       | charlie=w/alice     |                   |
(1 row)
```

Как видим, у пользователя `bob` пропало право передачи привилегии, а у пользователя `charlie` была отозвана и сама привилегия.

Аналогично можно отозвать по иерархии и привилегию:
```sql
REVOKE UPDATE ON alice.t1 FROM bob CASCADE;

REVOKE
```

```sql
\dp t1

                             Access privileges
 Schema | Name | Type  |  Access privileges  | Column privileges | Policies
--------+------+-------+---------------------+-------------------+----------
 alice  | t1   | table | alice=arwdDxt/alice+|                   |
        |      |       | bob=ardDxt/alice   +|                   |
        |      |       | charlie=w/alice     |                   |
(1 row)
```

## Функции

Роль `alice` создает простую функцию, возвращающую число строк в таблице `t2`:
```sql
CREATE FUNCTION f() RETURNS integer AS $$
  SELECT count(*)::integer FROM t2;
$$ LANGUAGE SQL;

CREATE FUNCTION
```

```sql
SELECT f();

 f 
---
 1
(1 row)
```

Роль `public` автоматически получает эту привилегию для любой создаваемой функции.
Поэтому, например, пользователь `bob` может выполнить функцию, которую только что создала роль `alice`.

Отчасти это компенсируется тем, что по умолчанию (или при явном указании `SECURITY INVOKER`)
функция выполняется с правами и в окружении вызывающей роли, включая путь поиска:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

```sql
SELECT alice.f();

ERROR:  relation "t2" does not exist
LINE 1:  SELECT count(*)::integer FROM t2;
                                       ^
QUERY:   SELECT count(*)::integer FROM t2;
CONTEXT:  SQL function "f" during inlining
```

Поэтому пользователь `bob` не сможет получить доступ к объектам, на которые ему не выданы привилегии.

Если же пользователь `bob` создаст свою таблицу `t2`, функция будет работать с этой таблицей для роли `bob`:
```sql
CREATE TABLE t2(n numeric);

CREATE TABLE
```

```sql
SELECT alice.f();

 f 
---
 0
(1 row)
```

Одна и та же функция выдает разный результат, потому что внутри идет обращение к разным объектам.

Другой доступный вариант - объявить функцию, как работающую с правами создавшего (`SECURITY DEFINER`):
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

```sql
CREATE OR REPLACE FUNCTION f() RETURNS integer AS $$
  SELECT count(*)::integer FROM t2;
$$ SECURITY DEFINER LANGUAGE SQL;

CREATE FUNCTION
```

В этом случае функция работает в контексте создавшей роли, независимо от того, кто ее вызывает:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

```sql
SELECT alice.f();

 f 
---
 1
(1 row)
```

В таком случае, конечно, надо внимательно следить за выданными привилегиями.
Скорее всего, потребуется отозвать привилегию `EXECUTE` у роли `public` и выдать ее явно только нужным ролям:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

```sql
REVOKE EXECUTE ON ALL FUNCTIONS IN SCHEMA alice FROM public;

REVOKE
```

```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

```sql
SELECT alice.f();

ERROR:  permission denied for function f
```

Дело осложняется тем, что привилегия на выполнение автоматически выдается роли `public` на каждую вновь создаваемую функцию,
и это поведение нельзя изменить:

```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

```sql
CREATE FUNCTION f_new() RETURNS integer AS $$ SELECT 1; $$ LANGUAGE SQL;

CREATE FUNCTION
```

```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

```sql
SELECT alice.f_new();

 f_new 
-------
     1
(1 row)
```

## Привилегии по умолчанию

Однако можно автоматически отзывать привилегию `EXECUTE` с помощью механизма привилегий по умолчанию:

```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE alice
REVOKE EXECUTE ON FUNCTIONS FROM public;

ALTER DEFAULT PRIVILEGES
```

```sql
\ddp

           Default access privileges
 Owner | Schema |   Type   | Access privileges
-------+--------+----------+-------------------
 alice |        | function | alice=X/alice
(1 row)
```

```sql
DROP FUNCTION f_new();

DROP FUNCTION
```

```sql
CREATE FUNCTION f_new() RETURNS integer AS $$ SELECT 1; $$ LANGUAGE SQL;

CREATE FUNCTION
```

```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

```sql
SELECT alice.f_new();

ERROR:  permission denied for function f_new
```

Аналогично можно настроить дополнительные правила для выдачи привилегий.
Пусть роль `alice` хочет, чтобы пользователь `bob` автоматически получал право чтения любой таблицы, которую она создает:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE alice
GRANT SELECT ON TABLES TO bob;

ALTER DEFAULT PRIVILEGES
```

```sql
\ddp

            Default access privileges
 Owner | Schema |   Type   |  Access privileges
-------+--------+----------+---------------------
 alice |        | function | alice=X/alice
 alice |        | table    | alice=arwdDxt/alice+
       |        |          | bob=r/alice
(2 rows)
```

```sql
CREATE TABLE t3(n integer);

CREATE TABLE
```

```sql
\dp t3

                             Access privileges
 Schema | Name | Type  |  Access privileges  | Column privileges | Policies
--------+------+-------+---------------------+-------------------+----------
 alice  | t3   | table | alice=arwdDxt/alice+|                   |
        |      |       | bob=r/alice         |                   |
(1 row)
```




## Табличные пространства

Привилегия `CREATE` разрешает создавать объекты в этом ТП.


## Последовательности

Выбирая нужные привилегии, можно разрешить или запретить доступ к трем управляющим функциям:
- `SELECT` - `currval`
- `UPDATE` - `nextval` и `setval`
- `USAGE` - `currval` и `nextval`
