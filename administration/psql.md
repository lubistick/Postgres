# Консольная утилита psql

Утилита `psql` - терминальный клиент для работы с СУБД `Postgres`. Это единственный клиент, поставляемый вместе с СУБД.


## Запуск

Команда для запуска `psql`:
```bash
psql -d <база_данных> -U <пользователь> -h <хост> -p <порт>
```

Запустим:
```bash
psql -U postgres

psql (14.7)
Type "help" for help.
```
Если параметры не указать, будут использованы значения по умолчанию:
- `d` - база данных (БД), совпадает с именем пользователя
- `U` - пользователь, совпадает с именем пользователя операционной системы (OC)
- `h` - хост, локальное соединение
- `p` - порт, `5432`

Поменялось приглашение. Мы запустили `psql`.


## Подключение к базе данных

Команда для подключения к БД:
```bash
\c <база_данных> <пользователь> <хост> <порт>
```

Подключимся к стандартной БД:
```bash
\c postgres

You are now connected to database "postgres" as user "postgres".
```

Посмотрим текущее подключение:
```bash
\conninfo

You are connected to database "postgres" as user "postgres" via socket in "/var/run/postgresql" at port "5432".
```


## Справка

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
\h
```

Синтаксис команды SQL:
```bash
\h <команда_SQL>
```


## Вывод результата запросов

Утилита `psql` умеет выводить результаты запросов в разных форматах.
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

По умолчанию ширина столбцов выравнена по самому широкому значению колонок.
Выводится строка с названиями столбцов - `schemaname`, `tablename`, `tableowner`.
Выводится количество строк в выборке - `(3 rows)`.

Текущие настройки форматирования:
```sql
\pset

border                   1      
columns                  0      
csv_fieldsep             ','    
expanded                 off    
fieldsep                 '|'    
fieldsep_zero            off    
footer                   on     
format                   aligned
linestyle                ascii  
null                     ''     
numericlocale            off    
pager                    1
pager_min_lines          0
recordsep                '\n'
recordsep_zero           off
tableattr
title
tuples_only              off
unicode_border_linestyle single
unicode_column_linestyle single
unicode_header_linestyle single
```


### Выравнивание

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

Расширенный режим можно установить только для одного запроса, если вместо `;` написать `\gx`:
```sql
SELECT * FROM pg_tables WHERE tablename = 'pg_proc' \gx

-[ RECORD 1 ]-----------
schemaname  | pg_catalog
tablename   | pg_proc
tableowner  | postgres
tablespace  |
hasindexes  | t
hasrules    | f
hastriggers | f
rowsecurity | f
```


### Разделитель столбцов

Зададим `пробел` в качестве разделителя между значениями столбцов:
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


## Команды операционной системы

В `psql` можно выполнять команды ОС:
```sql
\! whoami

root
```


## Переменные psql

Посмотрим все переменные:
```sql
\set

AUTOCOMMIT = 'on'                   
COMP_KEYWORD_CASE = 'preserve-upper'
DBNAME = 'postgres'                 
ECHO = 'none'                       
ECHO_HIDDEN = 'off'                 
ENCODING = 'UTF8'
ERROR = 'false'
FETCH_COUNT = '0'
HIDE_TABLEAM = 'off'
HIDE_TOAST_COMPRESSION = 'off'
HISTCONTROL = 'none'
HISTSIZE = '500'
HOST = '/var/run/postgresql'
IGNOREEOF = '0'
LAST_ERROR_MESSAGE = 'syntax error at or near "Hi"'
LAST_ERROR_SQLSTATE = '42601'
ON_ERROR_ROLLBACK = 'off'
ON_ERROR_STOP = 'off'
PORT = '5432'
PROMPT1 = '%/%R%x%# '
PROMPT2 = '%/%R%x%# '
PROMPT3 = '>> '
QUIET = 'off'
ROW_COUNT = '1'
SERVER_VERSION_NAME = '14.7'
SERVER_VERSION_NUM = '140007'
SHOW_CONTEXT = 'errors'
SINGLELINE = 'off'
SINGLESTEP = 'off'
SQLSTATE = '00000'
USER = 'postgres'
VERBOSITY = 'default'
VERSION = 'PostgreSQL 14.7 on x86_64-pc-linux-musl, compiled by gcc (Alpine 12.2.1_git20220924-r4) 12.2.1 20220924, 64-bit'
VERSION_NAME = '14.7'
VERSION_NUM = '140007'
```

Установим переменную `TEST`:
```sql
\set TEST Hi!
```

Выведем ее:
```sql
\echo :TEST

Hi!
```

Удалим переменную:
```sql
\unset TEST
```

Проверим:
```sql
\echo :TEST

:TEST
```

Запишем результат запроса в переменную:
```sql
SELECT now() AS cur_time \gset
```

Прочитаем:
```sql
\echo :cur_time

2024-05-13 23:35:11.848301+00
```
