# Привилегии


## Создание объектов

В нашем примере пользователь `alice` будет владельцем нескольких объектов в своей схеме.

Создадим роль `alice` с правом подключаться к БД:
```sql
CREATE ROLE alice LOGIN;

CREATE ROLE
```

Создадим схему:
```sql
CREATE SCHEMA alice;

CREATE SCHEMA
```

Дадим роли привилегии `CREATE` и `USAGE` на схему:
```sql
GRANT CREATE, USAGE ON SCHEMA alice TO alice;

GRANT
```

Подключимся под пользователем `alice`:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

Роль `alice` создает пару таблиц:
```sql
CREATE TABLE t1(n integer);

CREATE TABLE
```

```sql
CREATE TABLE t2(n integer, m integer);

CREATE TABLE
```

Объекты будут создаваться в схеме `alice`, т.к. по умолчанию схема равна имени пользователя.

```sql
\c - postgres

You are now connected to database "postgres" as user "postgres".
```

Создадим роль `bob` с правом подключаться к БД:
```sql
CREATE ROLE bob LOGIN;

CREATE ROLE
```

## Привилегии

Роль `bob` пробует обратиться к таблице `t1`:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

```sql
SELECT * FROM alice.t1;

ERROR:  permission denied for schema alice
LINE 1: SELECT * FROM alice.t1;
                      ^
```

В чем причина ошибки?

У пользователя `bob` нет доступа к схеме, т.к. он не суперпользователь, не владелец схемы, и не имеет нужных привилегий:
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

Подключимся под пользователем `alice`:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

Попробуем выдать привилегии пользователю `bob`:
```sql
GRANT CREATE, USAGE ON SCHEMA alice TO bob;

WARNING:  no privileges were granted for "alice"
GRANT
```

Почему привилегии не выдались?
Пользователь `alice` - не владелец схемы `alice`, поэтому выдавать привилегии не может.

Подключимся под пользователем `postgres`:
```sql
\c - postgres

You are now connected to database "postgres" as user "postgres".
```

Сделаем пользователя `alice` владельцем схемы `alice`:
```sql
ALTER SCHEMA alice OWNER TO alice;

ALTER SCHEMA
```

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

Теперь роль `alice` может выдать доступ роли `bob`:
```sql
GRANT CREATE, USAGE ON SCHEMA alice TO bob;

GRANT
```

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
