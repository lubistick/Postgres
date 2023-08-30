# Psql

Это терминальный клиент для работы с `Postgres`.

Поставляется вместе с СУБД.

Используется администраторами и разработчиками для интерактивной работы и выполнения скриптов.


## Использование

### Запуск

```bash
psql -d <database> -U <role> -h <host> -p <port>
```

Пример:

```bash
psql -U postgres

psql (14.7)
Type "help" for help.
```

Мы подключились к `psql`, поменялось приглашение. Теперь мы вводим не команды операционной системы, а команды сервера `Postgres`.


### Новое подключение к базе данных в psql

```bash
\c[onnect] <database> <role> <host> <port>
```

Пример:

```bash
\c demo

You are now connected to database "demo" as user "postgres".
```

О том как поднять демонстрационную базу данных смотрим [здесь](../optimization/README.md).


### Информация о текущем подключении

```bash
\conninfo
```

Пример:

```bash
\conninfo

You are connected to database "demo" as user "postgres" via socket in "/var/run/postgresql" at port "5432".
```


### Справка

Список команд `psql`:

```bash
\?
```

Системные переменные `psql`:

```bash
\? variables
```

Список команд SQL:

```bash
\h[elp]
```

Синтаксис команды SQL:

```bash
\h <sqlcommand>
```
