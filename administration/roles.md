# Роли и атрибуты

Роль - пользователь СУБД.
Роли являются общими объектами кластера, одна роль может подключаться к разным БД и быть владельцем объектов в разных БД.

Посмотрим все имеющиеся роли через системный каталог:
```sql
SELECT usename FROM pg_user;

 usename  
----------
 postgres
(1 row)
```

Или с помощью команды `\du`:
```sql
\du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

Роль `postgres` - суперпользователь, созданный при инициализации кластера.
У пользователя есть атрибуты, дающие право:
- `LOGIN` - подключаться
- `SUPERUSER` - права суперпользователя
- `CREATEDB` - создавать БД
- `CREATEROLE` - создавать/изменять других пользователей
- `REPLICATION` - использовать протокол репликации
- др.

Атрибут `LOGIN` не отображается в выводе команды `\du`, а его отсутствие отображается (было бы написано `Cannot login`).


## Создание роли

Создадим пользователя `alice` с возможностью подключаться и создавать другие роли:
```sql
CREATE ROLE alice LOGIN CREATEROLE;

CREATE ROLE
```

Подключимся под пользователем `alice`:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

Пользовать `alice` может создать новую роль, например `bob` с возможностью подключаться:
```sql
CREATE ROLE bob LOGIN;

CREATE ROLE
```

Подключимся под пользователем `bob`:
```sql
\c - bob

You are now connected to database "postgres" as user "bob".
```

Пользователь `bob` не может создать новую роль, т.к. у него нет соответствующих прав:
```sql
CREATE ROLE charlie LOGIN;

ERROR:  permission denied to create role
```


## Изменение атрибутов роли

Подключимся под пользователем `alice`:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

Пользователь `alice` может отобрать у роли `bob` право на вход.
Чтобы это сделать надо добавить приставку `NO` к имени атрибута:
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

Теперь пользователь `alice` не может создавать другие роли:
```sql
CREATE ROLE charlie LOGIN;

ERROR:  permission denied to create role
```

И изменять их атрибуты:
```sql
ALTER ROLE bob LOGIN;

ERROR:  permission denied
```


## Групповые роли

Роль может также выступать в качестве группы пользователей.
Включим пользователя `alice` в группу `postgres`.
Другими словами, разрешим роли `alice` действовать от имени супервользовательской роли `postgres`.
Это напоминает команду `su` в ОС `Unix`.

Подключимся как пользователь `postgres`:
```sql
\c - postgres

You are now connected to database "postgres" as user "postgres".
```

Включим пользователя `alice` в группу `postgres` с помощью команды `GRANT`:
```sql
GRANT postgres TO alice;

GRANT ROLE
```

Посмотрим имеющиеся роли:
```sql
\du

                                    List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+------------
 alice     |                                                            | {postgres}
 bob       | Cannot login                                               | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

Теперь пользователь `alice` член группы `postgres` (колонка `Member of`).
Подключимся под пользователем `alice`:
```sql
\c - alice

You are now connected to database "postgres" as user "alice".
```

Роль `alice` не получает возможности групповой роли автоматически.
Она может ими воспользоваться, только если переключится на эту роль командой `SET ROLE`:
```sql
SET ROLE postgres;

SET
```

```sql
ALTER ROLE bob LOGIN;

ALTER ROLE
```

Чтобы понять, кем является пользователь на самом деле и на какую роль он переключился, есть функции:
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


## Установка конфигурационных параметров для роли

Чтобы пользователь `alice` не злоупотреблял полномочиями, сделаем так, чтобы все его команды попадали в журнал сообщений.
Это еще один вариант установки конфигурационных параметров - он сработает при подключении пользователя `alice` к серверу.
```sql
ALTER ROLE alice SET log_min_duration_statement = 0;

ALTER ROLE
```

Можно ограничить действия роли в конкретной БД:
```sql
ALTER ROLE alice RESET log_min_duration_statement;

ALTER ROLE
```

```sql
ALTER ROLE alice IN DATABASE postgres SET log_min_duration_statement = 0;

ALTER ROLE
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
