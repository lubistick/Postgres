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
