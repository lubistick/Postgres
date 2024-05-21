# Роли и атрибуты


## Создание ролей

В этом модуле приглашение в терминале будет показывать роль, от имени которой выполняется команда.

Создадим роль:
```sql
postgres=# CREATE ROLE alice LOGIN CREATEROLE;

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

Пользовать `alice` может создать нового пользователя:
```sql
CREATE ROLE bob LOGIN;

CREATE ROLE
```


