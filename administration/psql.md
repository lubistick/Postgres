# Psql

Это терминальный клиент для работы с `Postgres`.

Поставляется вместе с СУБД.

Используется администраторами и разработчиками для интерактивной работы и выполнения скриптов.


## Основы

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


### Подключение к базе данных в psql

```bash
\c[onnect] <database> <role> <host> <port>
```

Пример:
```bash
\c demo

You are now connected to database "demo" as user "postgres".
```

О том как поднять демонстрационную базу данных смотрим [здесь](../optimization/demo.md).


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


## Вывод результата запросов

`Psql` умеет выводить результаты запросов в разных форматах.


### Выравнивание

Сделаем запрос, например:
```sql
SELECT schemaname, tablename, tableowner FROM pg_tables LIMIT 3;

 schemaname |    tablename     | tableowner
------------+------------------+------------
 pg_catalog | pg_statistic     | postgres
 pg_catalog | pg_type          | postgres
 pg_catalog | pg_foreign_table | postgres
(3 rows)
```

По умолчанию вывод с выравниванием.

Переключим в режим без выравнивания:
```bash
\a

Output format is unaligned.
```

Смотрим вывод:
```sql
SELECT schemaname, tablename, tableowner FROM pg_tables LIMIT 3;

schemaname|tablename|tableowner
pg_catalog|pg_statistic|postgres
pg_catalog|pg_type|postgres
pg_catalog|pg_foreign_table|postgres
(3 rows)
```

Переключим обратно в режим с выравниванием:
```bash
\a

Output format is aligned.
```

### Шапка и итоговая строка

Переключим в режим без отображения шапки таблицы и итоговой строки:
```bash
\t

Tuples only is on.
```

Смотрим вывод:
```sql
SELECT schemaname, tablename, tableowner FROM pg_tables LIMIT 3;

pg_catalog | pg_statistic     | postgres
pg_catalog | pg_type          | postgres
pg_catalog | pg_foreign_table | postgres
```

Выводятся только кортежи.

Переключим обратно в режим с отображением шапки таблицы и итоговой строки:
```bash
\t

Tuples only is off.
```

### Разделитель столбцов

Зададим разделитель между значениями столбцов:
```bash
\pset fieldsep ' '

Field separator is " ".
```

Это работает только в режиме без выравнивания:

```sql
SELECT schemaname, tablename, tableowner FROM pg_tables LIMIT 3;

schemaname tablename tableowner
pg_catalog pg_statistic postgres
pg_catalog pg_type postgres
pg_catalog pg_foreign_table postgres
(3 rows)
```

### Расширенный режим

Включим расширенный режим:
```bash
\x

Expanded display is on.
```

Выполним запрос, например:
```sql
SELECT * FROM pg_tables WHERE tablename = 'pg_class';

-[ RECORD 1 ]-----------
schemaname  | pg_catalog
tablename   | pg_class
tableowner  | postgres
tablespace  |
hasindexes  | t
hasrules    | f
hastriggers | f
rowsecurity | f

```
Вывод как бы повернулся на бок - в первом столбце названия колонок, во втором их значения.

Расширенный формат удобен, когда нужно вывести малое количество записей, которые в обычном режиме не влазят в экран.

Переключим обратно в обычный режим:
```bash
\x

Expanded display is off.
```
