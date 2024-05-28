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

