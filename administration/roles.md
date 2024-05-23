# Роли и атрибуты


## Создание ролей

Создадим роль:
```sql
CREATE ROLE alice LOGIN CREATEROLE;

CREATE ROLE
```

Мы создали роль `alice`, которая имеет возможность:
- `LOGIN` - подключаться
- `CREATEROLE` - создавать друге роли

Подключимся как пользователь `alice`:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

Пользовать `alice` может создать нового пользователя `bob`:
```sql
CREATE ROLE bob LOGIN;

CREATE ROLE
```

Подключимся как пользователь `bob`:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

Пользователь `bob` не может создать нового пользователя:
```sql
CREATE ROLE charlie LOGIN;

ERROR:  permission denied to create role
```

Посмотрим роли, имеющиеся в кластере командой `\du`:
```sql
\du

                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 alice     | Create role                                                | {}
 bob       |                                                            | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

Или в системном каталоге:
```sql
SELECT usename FROM pg_user;
 usename  
----------
 postgres
 alice
 bob
(3 rows)
```

Существующие роли можно изменять. Пользователь `alice` может отобрать у пользователя `bob` право на вход:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

```sql
ALTER ROLE bob NOLOGIN;

ALTER ROLE
```

Попробуем войти под ролью `bob`:
```sql
\c - bob

connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  role "bob" is not permitted to log in
Previous connection kept
```

Пользователь `alice` может отобрать у себя самого право создавать другие роли:
```sql
ALTER ROLE alice NOCREATEROLE;

ALTER ROLE
```

Проверим:
```sql
CREATE ROLE charlie LOGIN;

ERROR:  permission denied to create role
```

Такие пары как `LOGIN` - `NOLOGIN` или `CREATEROLE` - `NOCREATEROLE` есть и у других атрибутов.

Теперь пользователь `alice` не может изменять атрибуты существующих ролей:
```sql
ALTER ROLE bob LOGIN;

ERROR:  permission denied
```


## Групповые роли

Чтобы наделить пользователя `alice` супервозможностями, включим ее в супервользовательскую роль `postgres`:

```sql
\c - postgres

You are now connected to database "postgres" as user "postgres".
```

```sql
GRANT postgres TO alice;

GRANT ROLE
```

```sql
\du

                                    List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+------------
 alice     |                                                            | {postgres}
 bob       | Cannot login                                               | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

Чтобы пользователь `alice` не злоупотреблял полномочиями, сделаем так, чтобы все его команды попадали в журнал сообщений:
```sql
ALTER ROLE alice SET log_min_duration_statement = 0;

ALTER ROLE
```

Это еще один вариант установки конфигурационных параметров - он сработает при подключении пользователя `alice` к серверу.

Можно ограничить действия и конкретной БД:
```sql
ALTER ROLE alice RESET log_min_duration_statement;

ALTER ROLE
```

```sql
ALTER ROLE alice IN DATABASE postgres SET log_min_duration_statement = 0;

ALTER ROLE
```

Роль `alice` не получает возможности групповой роли автоматически.
Она может ими воспользоваться, только если переключится на эту роль:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

```sql
SET ROLE postgres;

SET
```

```sql
ALTER ROLE bob LOGIN;

ALTER ROLE
```

Это напоминает команду `su` в ОС `Unix`.

Чтобы понять, кем является пользователь на самом деле и на какую роль он переключился, есть функция:
```sql
SELECT session_user, current_user;

 session_user | current_user 
--------------+--------------
 alice        | postgres
(1 row)
```

Вернемся к прежней роли:
```sql
RESET ROLE;

RESET
```

```sql
SELECT session_user, current_user;

 session_user | current_user 
--------------+--------------
 alice        | alice
(1 row)
```


## Владение объектами

Когда пользователь `alice` создает какой-либо объект в БД, он становится его владельцем:
```sql
CREATE TABLE test(id integer);

CREATE TABLE
```

Как в этом убедиться? Владелец указан в столбце `owner`:
```sql
\dt test

       List of relations
 Schema | Name | Type  | Owner
--------+------+-------+-------
 public | test | table | alice
(1 row)
```


## Удаление ролей

Удалить роль можно, если нет объектов, которыми она владеет.

```sql
\c - postgres

You are now connected to database "postgres" as user "postgres".
```

```sql
DROP ROLE alice;

ERROR:  role "alice" cannot be dropped because some objects depend on it
DETAIL:  owner of table test
```

Можно передать объекты другому пользователю:
```sql
REASSIGN OWNED BY alice TO bob;

REASSIGN OWNED
```

```sql
\dt test

       List of relations
 Schema | Name | Type  | Owner
--------+------+-------+-------
 public | test | table | bob
(1 row)
```

```sql
DROP ROLE alice;

DROP ROLE
```

Другой вариант - удалить все объекты:
```sql
DROP OWNED BY bob;

DROP OWNED
```

```sql
DROP ROLE bob;

DROP ROLE
```

Надо только иметь в виду, что роль может владеть объектами в разных БД.
