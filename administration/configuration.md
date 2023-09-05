# Конфигурирование

Основной файл конфигурации `postgresql.conf` читается один раз при старте сервера.

Если параметр указан несколько раз, применяется последнее значение.


## Postgresql.conf

Располагается в каталоге `PGDATA`:
```bash
echo $PGDATA

/var/lib/postgresql/data
```

Узнаем другим способом путь к файлу. Подключимся через `psql` и наберем команду:
```sql
SHOW config_file;

               config_file
------------------------------------------
 /var/lib/postgresql/data/postgresql.conf
(1 row)
```

Или такую:
```sql
SELECT setting FROM pg_settings WHERE name = 'config_file';

                 setting
------------------------------------------
 /var/lib/postgresql/data/postgresql.conf
(1 row)
```

Посмотрим первые 10 строк `postgresql.conf`:
```bash
head /var/lib/postgresql/data/postgresql.conf
# -----------------------------
# PostgreSQL configuration file
# -----------------------------
#
# This file consists of lines of the form:
#
#   name = value
#
# (The "=" is optional.)  Whitespace may be used.  Comments are introduced with
# "#" anywhere on a line.  The complete list of parameter names and allowed
```


### Pg_file_settings

В `postgresql.conf` много комментариев. Получим все незакомментированные строки:
```sql
SELECT sourceline, name, setting, applied FROM pg_file_settings WHERE sourcefile LIKE '%postgresql.conf';

 sourceline |            name            |      setting       | applied
------------+----------------------------+--------------------+---------
         60 | listen_addresses           | *                  | t
         65 | max_connections            | 100                | t
        127 | shared_buffers             | 128MB              | t
        150 | dynamic_shared_memory_type | posix              | t
        240 | max_wal_size               | 1GB                | t
        241 | min_wal_size               | 80MB               | t
        580 | log_timezone               | UTC                | t
        694 | datestyle                  | iso, mdy           | t
        696 | timezone                   | UTC                | t
        710 | lc_messages                | en_US.utf8         | t
        712 | lc_monetary                | en_US.utf8         | t
        713 | lc_numeric                 | en_US.utf8         | t
        714 | lc_time                    | en_US.utf8         | t
        717 | default_text_search_config | pg_catalog.english | t
(14 rows)
```

`applied` - значение параметра может быть применено без перезапуска сервера (да/нет).

Посмотрим строчки с 60-й по 65-ю:
```bash
\! sed -n '60,65p' $PGDATA/postgresql.conf

listen_addresses = '*'
                                        # comma-separated list of addresses;
                                        # defaults to 'localhost'; use '*' for all
                                        # (change requires restart)
#port = 5432                            # (change requires restart)
max_connections = 100                   # (change requires restart)
```

### Pg_settings

Получим действующие значения всех параметров в представлении `pg_settings`. Посмотрим значения параметра `work_mem`:
```sql
SELECT * FROM pg_settings WHERE name = 'work_mem'\gx

-[ RECORD 1 ]---+----------------------------------------------------------------------------------------------------------------------
name            | work_mem
setting         | 4096
unit            | kB
category        | Resource Usage / Memory
short_desc      | Sets the maximum memory to be used for query workspaces.
extra_desc      | This much memory can be used by each internal sort operation and hash table before switching to temporary disk files.
context         | user
vartype         | integer
source          | default
min_val         | 64
max_val         | 2147483647
enumvals        |
boot_val        | 4096
reset_val       | 4096
sourcefile      |
sourceline      |
pending_restart | f
```
Значение колонок `pg_settings`:
- `boot_val` - значение по умолчанию. Если значение закомментировано (не задано) в конфигурационных файлах - при запуске сервера будет взято отсюда.
- `reset_val` - если изменить параметр во время сеанса, а потом набрать команду `RESET` - восстановится это значение.
- `sourre`, `sourcefile`, `sourceline` - источник текущего значения параметра. Значение `default` - значения взято из `boot_val`.
- `pending_restart` - требуется перезапуск сервера, чтобы вступило в силу новое значение параметра (да/нет).
- `context` - действия, необходимые для применения параметра, возможные значения:
    - `internal` - изменить нельзя, задано при установке (например, параметр `lc_collate`, потому что завязан на правилах сортировки которое используется в базе данных)
    - `postmaster` - требуется перезапуск сервера
    - `sighup` - требуется перечитать файлы конфигурации
    - `superuser` - суперпользователь может изменить для своего сеанса
    - `user` - любой пользователь может изменить для своего сеанса

### Редактирование

Запишем один и тот же параметр в `postgresql.conf` несколько раз.
```bash
\! echo work_mem=12MB >> /var/lib/postgresql/data/postgresql.conf
\! echo work_mem=8MB >> /var/lib/postgresql/data/postgresql.conf
```

Может быть применено только значение 8Мб (колонка `applied`), т.к. оно находится ниже в файле, чем 12Мб:
```sql
SELECT * FROM pg_file_settings WHERE name = 'work_mem';

                sourcefile                | sourceline | seqno |   name   | setting | applied | error
------------------------------------------+------------+-------+----------+---------+---------+-------
 /var/lib/postgresql/data/postgresql.conf |        797 |    15 | work_mem | 12MB    | f       |
 /var/lib/postgresql/data/postgresql.conf |        798 |    16 | work_mem | 8MB     | t       |
(2 rows)
```

Перечитаем файл конфигурации, чтобы применились изменения:
```sql
SELECT pg_reload_conf();

 pg_reload_conf
----------------
 t
(1 row)
```

Смотрим `pg_settings`, изменились колонки со значением параметра (`setting`, `reset_val`) и источником (`source`, `sourcefile`, `sourceline`):
```sql
SELECT * FROM pg_settings WHERE name = 'work_mem'\gx

-[ RECORD 1 ]---+----------------------------------------------------------------------------------------------------------------------
name            | work_mem
setting         | 8192
unit            | kB
category        | Resource Usage / Memory
short_desc      | Sets the maximum memory to be used for query workspaces.
extra_desc      | This much memory can be used by each internal sort operation and hash table before switching to temporary disk files.
context         | user
vartype         | integer
source          | configuration file
min_val         | 64
max_val         | 2147483647
enumvals        |
boot_val        | 4096
reset_val       | 8192
sourcefile      | /var/lib/postgresql/data/postgresql.conf
sourceline      | 798
pending_restart | f
```

## Postgresql.auto.conf

Есть еще один файл конфигурации `postgresql.auto.conf`:
```bash
cat /var/lib/postgresql/data/postgresql.auto.conf

# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
```

Файл `postgresql.auto.conf` считывается после `postgresql.conf`. Поэтому в `postgresql.auto.conf` можно переопределить параметры, которые заданы в `postgresql.conf`.


### Редактирование

Чтобы редактировать файл `postgresql.auto.conf`, подключимся через `psql`.

Установим значение `work_mem` в 16Мб:
```sql
ALTER SYSTEM SET work_mem TO '16MB';
    
ALTER SYSTEM
```

В файл `postgresql.auto.conf` добавилась строка:
```bash
\! cat /var/lib/postgresql/data/postgresql.auto.conf

# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
work_mem = '16MB'
```

Изменились значения и в `pg_settings`.

Посмотрим значение `work_mem` в текущем сеансе:
```sql
SHOW work_mem;

 work_mem
----------
 8MB
(1 row)
```

Значение в 16Мб не было применено. Перечитаем конфигурацию:
```sql
SELECT pg_reload_conf();

 pg_reload_conf
----------------
 t
(1 row)
```

Значение применилось:
```sql
SHOW work_mem;

 work_mem
----------
 16MB
(1 row)
```

Удалим строку про `work_mem` из `postgresql.auto.conf`:
```sql
ALTER SYSTEM RESET work_mem;
    
ALTER SYSTEM
```

Чтобы удалить все строки:
```sql
ALTER SYSTEM RESET ALL;

ALTER SYSTEM
```

## Установка параметров для текущего сеанса

Поменяем значение `work_mem` прямо во время выполнения запросов:
```sql
SET work_mem TO '24MB';

SET
```

Или с помощью функции:
```sql
SELECT set_config('work_mem', '32MB', false);

 set_config
------------
 32MB
(1 row)
```

Третий параметр:
- `true` - установить значение только на время текущей транзакции
- `false` - установить значение до конца сеанса

Это важно при работе через пул соединений, когда в одном сеансе могут выполняться транзакции разных пользователей.

Получим значение параметра внутри запроса функцией `current_setting()`, т.к. внутри запроса мы не можем использовать команду `SHOW`:
```sql
SELECT current_setting('work_mem');

 current_setting
-----------------
 32MB
(1 row)
```

Команды установки параметров для текущего сеанса - транзакционные. Если транзакция завершится с ошибкой, значения параметров откатятся:

```sql
BEGIN;

BEGIN
```

```sql
SET work_mem TO '64MB';

SET
```

```sql
SHOW work_mem;

 work_mem
----------
 64MB
(1 row)
```

```sql
ROLLBACK;

ROLLBACK
```

```sql
SHOW work_mem;

 work_mem
----------
 32MB
(1 row)
```
