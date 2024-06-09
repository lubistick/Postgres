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
- `владелец объекта` - изначально получает полный набор привилегий
- `остальные роли` - изначально не имеют привилегий


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
- `U` - `usage`
- `C` - `create`

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

Роль `bob` снова пытается прочитать таблицу:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

```sql
SELECT * FROM alice.t1;

ERROR:  permission denied for table t1
```

В чем причина ошибки на этот раз?
Сейчас у пользователя `bob` есть доступ к схеме, но нет доступа к самой таблице:
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

Выдадим роли `bob` право на чтение:
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

Привилегии для таблиц:
- `a` - `INSERT`
- `r` - `SELECT`
- `w` - `UPDATE`
- `d` - `DELETE`
- `D` - `TRUNCATE`
- `x` - `REGERENCE`
- `t` - `TRIGGER`

Подключимся как пользователь `bob`:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

На этот раз у роли `bob` все получается:
```sql
SELECT * FROM alice.t1;

 n 
---
(0 rows)
```

А, например, вставить строку в таблицу он не сможет:
```sql
INSERT INTO alice.t1 VALUES (42);

ERROR:  permission denied for table t1
```

Некоторые привилегии можно выдать на определенные столбцы:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

```sql
GRANT INSERT(n, m) ON t2 TO bob;

GRANT
```

```sql
GRANT SELECT(m) ON t2 TO bob;

GRANT
```

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

А читать сможет только один столбец:
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

Если необходимо, роль `alice` может выдать пользователю `bob` все привилегии, не перечисляя их явно:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

```sql
GRANT ALL ON t1 TO bob;

GRANT
```

```sql
\dp t1

                             Access privileges
 Schema | Name | Type  |  Access privileges  | Column privileges | Policies
--------+------+-------+---------------------+-------------------+----------
 alice  | t1   | table | alice=arwdDxt/alice+|                   |
        |      |       | bob=arwdDxt/alice   |                   |
(1 row)
```

Теперь роли `bob` доступны все действия, например, удаление строк:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

```sql
DELETE FROM alice.t1;

DELETE 0
```

А удаление самой таблицы?
```sql
DROP TABLE alice.t1;

ERROR:  must be owner of table t1
```

Удалить таблицу может только владелец или суперпользователь, специальной привилегии для этого не существует.

## Групповые привилегии

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

