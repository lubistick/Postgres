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
- `владелец объекта` - изначально получает полный набор привилегий, а также удаление объекта, выдачу и отзыв привилегий на объект
- `остальные роли` - доступ в рамках выданных привилегий


## Скрытая роль public

Роль `public` не отображается в списке ролей, но она существует.
Все другие роли неявно входят в групповую роль `public`, а значит, имеют ее привилегии.
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
  - `EXECUTE` права выполнять функцию


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
В колонке `Access privileges` хранятся выданные привилегии (сейчас она пуста):
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
Но он все равно может подключиться к БД, т.к. любой пользователь входит в групповую роль `public`, у которой эта привилегия есть:
```sql
\c - application

You are now connected to database "access_privileges" as user "application".
```

А вот создать схему внутри БД не может:
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

Дадим пользователю `application` привилегию на создание схем внутри БД `access_privileges`:
```sql
GRANT CREATE ON DATABASE access_privileges TO application;

GRANT
```

Посмотрим информацию о БД:
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

Смотрим третью подстроку:
пользователь `application` получил привилегию на создание схем `C` от суперпользователя `postgres`.

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
- `C` - `CREATE` - разрешает создавать объекты в этой схеме
- `U` - `USAGE` - разрешает обращаться к объектам этой схемы

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

У пользователя `microservice` нет никаких привилегии обращаться к объектам схемы `application`.
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

Выдадим все привилегии на схему `application` пользователю `microservice`, не перечисляя их явно:
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












В нашем примере роль `alice` будет владельцем нескольких объектов в своей схеме.
Подключимся под суперпользователем `postgres`:
```sql
\c - postgres

You are now connected to database "postgres" as user "postgres".
```

Создадим роль `alice` с правом подключаться к БД:
```sql
CREATE ROLE alice LOGIN;

CREATE ROLE
```

Создадим пользователя `bob` с правом подключаться к БД:
```sql
CREATE ROLE bob LOGIN;

CREATE ROLE
```

Создадим схему `alice`:
```sql
CREATE SCHEMA alice;

CREATE SCHEMA
```

Дадим роли `alice` привилегии `CREATE` и `USAGE` на схему `alice`:
```sql
GRANT CREATE, USAGE ON SCHEMA alice TO alice;

GRANT
```

Подключимся под ролью `alice`:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

Создадим таблицу `t1` с одной колонкой.
Она будет создана в схеме `alice`, т.к. по умолчанию схема равна имени пользователя:
```sql
CREATE TABLE t1(n integer);

CREATE TABLE
```

Создадим таблицу `t2` с двумя колонками:
```sql
CREATE TABLE t2(n integer, m integer);

CREATE TABLE
```


## Схемы

Подключимся под пользователем `bob`:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

Прочитаем таблицу `t1` в схеме `alice`:
```sql
SELECT * FROM alice.t1;

ERROR:  permission denied for schema alice
LINE 1: SELECT * FROM alice.t1;
                      ^
```

Ошибка! У пользователя `bob` нет доступа к схеме `alice`, т.к. он:
- не суперпользователь
- не владелец схемы
- не имеет нужных привилегий

Посмотрим описание схемы `alice`:
```sql
\dn+ alice

                    List of schemas
 Name  |  Owner   |  Access privileges   | Description
-------+----------+----------------------+-------------
 alice | postgres | postgres=UC/postgres+|
       |          | alice=UC/postgres    |
(1 row)
```

Колонка `Access privileges` содержит две подстроки, разделенные переносом.
Каждая подстрока представлена в формате: `<роль>=<привилегии>/<кем_выданы>`.
Названия привилегий обозначаются одной буквой.
Привилегии для схем:
- `U` - `USAGE` - разрешает обращаться к объектам этой схемы
- `C` - `CREATE` - разрешает создавать объекты в этой схеме

Подключимся под ролью `alice`:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

Выдадим привилегии пользователю `bob`:
```sql
GRANT CREATE, USAGE ON SCHEMA alice TO bob;

WARNING:  no privileges were granted for "alice"
GRANT
```

Ошибка! Роль `alice` - не владелец схемы `alice`, поэтому выдавать привилегии не может.
Подключимся под суперпользователем `postgres`:
```sql
\c - postgres

You are now connected to database "postgres" as user "postgres".
```

Сделаем роль `alice` владельцем схемы `alice`:
```sql
ALTER SCHEMA alice OWNER TO alice;

ALTER SCHEMA
```

Посмотрим описание схемы `alice`. Значение в колонке `Owner` изменилось:
```sql
\dn+ alice

                 List of schemas
 Name  | Owner | Access privileges | Description
-------+-------+-------------------+-------------
 alice | alice | alice=UC/alice    |
(1 row)
```

Подключимся под пользователем `alice`:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

Теперь роль `alice` может выдать доступ пользователю `bob`:
```sql
GRANT CREATE, USAGE ON SCHEMA alice TO bob;

GRANT
```


## Таблицы

Больше всего привилегий определено для таблиц:
- `a` - append - `INSERT`
- `r` - read - `SELECT`
- `w` - write - `UPDATE`
- `d` - delete - `DELETE`
- `D` - delete - `TRUNCATE`
- `x` - foreign keys - `REFERENCES`
- `t` - `TRIGGER`

- Подключимся под пользователем `bob`:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

Снова прочитаем таблицу `t1` в схеме `alice`:
```sql
SELECT * FROM alice.t1;

ERROR:  permission denied for table t1
```

Опять ошибка!
Сейчас у пользователя `bob` есть доступ к схеме, но нет доступа к самой таблице.
Посмотрим привилегии у таблицы:
```sql
\dp alice.t1

                            Access privileges
 Schema | Name | Type  | Access privileges | Column privileges | Policies
--------+------+-------+-------------------+-------------------+----------
 alice  | t1   | table |                   |                   |
(1 row)
```
Пустое поле `Access privileges` означает, что у владельца есть полный набор привилегий, а кроме него никто не имеет доступа.

Подключимся как пользователь `alice`:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

Дадим пользователю `bob` право на чтение (`SELECT`):
```sql
GRANT SELECT ON t1 TO bob;

GRANT
```

Посмотрим, как изменились привилегии:
```sql
\dp t1

                             Access privileges
 Schema | Name | Type  |  Access privileges  | Column privileges | Policies
--------+------+-------+---------------------+-------------------+----------
 alice  | t1   | table | alice=arwdDxt/alice+|                   |
        |      |       | bob=r/alice         |                   |
(1 row)
```

Подключимся под пользователем `bob`:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

Теперь пользователь `bob` может читать таблицу `t1` в схеме `alice`:
```sql
SELECT * FROM alice.t1;

 n 
---
(0 rows)
```

Но вставить строку в таблицу пользователь `bob` не сможет:
```sql
INSERT INTO alice.t1 VALUES (42);

ERROR:  permission denied for table t1
```


### Все привилегии таблицы

Подключимся под ролью `alice`:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

Роль `alice` может выдать пользователю `bob` все привилегии, не перечисляя их явно:
```sql
GRANT ALL ON t1 TO bob;

GRANT
```

Посмотрим привилегии для таблицы `t1`:
```sql
\dp t1

                             Access privileges
 Schema | Name | Type  |  Access privileges  | Column privileges | Policies
--------+------+-------+---------------------+-------------------+----------
 alice  | t1   | table | alice=arwdDxt/alice+|                   |
        |      |       | bob=arwdDxt/alice   |                   |
(1 row)
```

Подключимся под пользователем `bob`:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

Роли `bob` доступны все действия, например, удаление строк:
```sql
DELETE FROM alice.t1;

DELETE 0
```

Удалить таблицу может только владелец или суперпользователь, привилегии для этого не существует:
```sql
DROP TABLE alice.t1;

ERROR:  must be owner of table t1
```


### Колонки таблицы

Некоторые привилегии можно выдать на определенные колонки.
Подключимся под ролью `alice`:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

Дадим пользователю `bob` право на вставку в колонки `n` и `m` в таблице `t2`:
```sql
GRANT INSERT(n, m) ON t2 TO bob;

GRANT
```

И право читать только колонку `m`:
```sql
GRANT SELECT(m) ON t2 TO bob;

GRANT
```

Посмотрим привилегии таблицы `t2` - в столбце `Column privileges` появились соответствующие права доступа:
```sql
\dp t2

                            Access privileges
 Schema | Name | Type  | Access privileges | Column privileges | Policies
--------+------+-------+-------------------+-------------------+----------
 alice  | t2   | table |                   | n:               +|
        |      |       |                   |   bob=a/alice    +|
        |      |       |                   | m:               +|
        |      |       |                   |   bob=ar/alice    |
(1 row)
```

Подключимся под пользователем `bob`:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

Теперь пользователь `bob` может добавлять строки в таблицу `t2`:
```sql
INSERT INTO alice.t2 VALUES(1, 2);

INSERT 0 1
```

А читать может только столбец `m`:
```sql
SELECT * FROM alice.t2;

ERROR:  permission denied for table t2
```

```sql
SELECT m FROM alice.t2;

 m 
---
 2
(1 row)
```


## Роль public



Пусть роль `alice` выдаст роли `public` (описать паблик!) привилегию изменения t2:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

Роль `public` - групповая роль, в которую включены абсолютно все.

```sql
GRANT UPDATE ON t2 TO public;

GRANT
```

```sql
\dp t2

                             Access privileges
 Schema | Name | Type  |  Access privileges  | Column privileges | Policies
--------+------+-------+---------------------+-------------------+----------
 alice  | t2   | table | alice=arwdDxt/alice+| n:               +|
        |      |       | =w/alice            |   bob=a/alice    +|
        |      |       |                     | m:               +|
        |      |       |                     |   bob=ar/alice    |
(1 row)
```

Пустая роль слева от знака равенства обозначает `public`.

Проверим, сможет ли роль `bob` воспользоваться этой привилегией:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

```sql
UPDATE alice.t2 SET n = n + 1;

ERROR:  permission denied for table t2
```

В чем причина ошибки?
Дело в том, что перед обновлением t2 сначала необходимо выбрать нужные строки, а для этого требуется привилегия чтения
(как минимум столбца `n`, который используется в условии).
Но пользователь `bob` имеет право читать только столбец `m`.

```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

```sql
GRANT SELECT ON t2 TO bob;

GRANT
```

```sql
\dp t2

                             Access privileges
 Schema | Name | Type  |  Access privileges  | Column privileges | Policies
--------+------+-------+---------------------+-------------------+----------
 alice  | t2   | table | alice=arwdDxt/alice+| n:               +|
        |      |       | =w/alice           +|   bob=a/alice    +|
        |      |       | bob=r/alice         | m:               +|
        |      |       |                     |   bob=ar/alice    |
(1 row)
```

Вот теперь у пользователя `bob` появилась возможность обновления таблицы:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

```sql
UPDATE alice.t2 SET n = n + 1;

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
