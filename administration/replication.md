# Физическая репликация

Репликация - это синхронизация нескольких копий кластера баз данных на разных серверах.
Репликация решает две задачи: отказоустойчивость и масштабируемость.

При физической репликации серверы имеют назначенные роли: мастер и реплика.
Мастер передает журнальные записи на реплику, та их применяет к своим файлам.
Важна двоичная совместимость между серверами (одинаковые платформы и основные версии Postgres).
Реплицировать можно только весь кластер целиком.

Развернем реплику из физической копии, используем утилиту `pg_basebackup`:

```bash
pg_basebackup -U postgres -h postgres-main --pgdata=/home/basebackup -R --checkpoint=fast
```

- ключ `checkpoint` со значением `fast` просит утилиту выполнить контрольную точку как можно быстрее
- ключ `R` добавляет настройки для реплики

Утилита создала сигнальный файл (ключ `-R`), который даст реплике указание перейти в режим постоянного восстановления:

```bash
ls -l /home/basebackup/standby.signal

-rw-------    1 root     root             0 Dec 15 15:56 /home/basebackup/standby.signal
```

Копия была размещена в домашнем каталоге, перенесем ее в каталог данных кластера:

```bash
rm -rf $PGDATA
```

```bash
mv /home/basebackup $PGDATA
```

Делаем владельцем файлов пользователя "postgres"
```bash
chown -R postgres: $PGDATA
```

Остановим контейнер с репликой:
```bash
docker compose stop postgres-replica
```

Закомменитруем `# command: ["sleep", "infinity"]` в `docker-compose.yaml` и запустим реплику:
```bash
docker compose up postgres-replica -d
```

Посмотрим процессы на реплике:
```bash
ps

PID   USER     TIME  COMMAND
    1 postgres  0:00 postgres
   27 postgres  0:00 postgres: io worker 0
   28 postgres  0:00 postgres: io worker 1
   29 postgres  0:00 postgres: io worker 2
   30 postgres  0:00 postgres: checkpointer
   31 postgres  0:00 postgres: background writer
   32 postgres  0:00 postgres: startup recovering 000000010000000000000003
   33 postgres  0:00 postgres: walreceiver
   34 root      0:00 sh
   40 root      0:00 ps
```

- процесс `walreceiver` принимает поток журнальных записей
- процесс `startup` применяет изменения

Посмотрим процессы на мастере:
```bash
ps
PID   USER     TIME  COMMAND
    1 postgres  0:00 postgres
   27 postgres  0:00 postgres: io worker 1
   28 postgres  0:00 postgres: io worker 0
   29 postgres  0:00 postgres: io worker 2
   30 postgres  0:00 postgres: checkpointer
   31 postgres  0:00 postgres: background writer
   33 postgres  0:00 postgres: walwriter
   34 postgres  0:00 postgres: autovacuum launcher
   35 postgres  0:00 postgres: logical replication launcher
   43 root      0:00 sh
  110 postgres  0:00 postgres: walsender postgres 172.28.0.3(55636) streaming 0/3000168
  145 root      0:00 ps
```
- процесс `walsender` отправляет реплике журнальные записи


Состояние реплики можно смотреть в специальном представлении на мастере:
```sql
select * from pg_stat_replication \gx

-[ RECORD 1 ]----+------------------------------
pid              | 110
usesysid         | 10
usename          | postgres
application_name | walreceiver
client_addr      | 172.28.0.3
client_hostname  | 
client_port      | 55636
backend_start    | 2025-12-15 16:21:03.221467+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/3000168
write_lsn        | 0/3000168
flush_lsn        | 0/3000168
replay_lsn       | 0/3000168
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2025-12-15 16:23:13.269491+00
```


## Проверка реплики

Создадим БД на мастере:
```bash
CREATE DATABASE replica_overview;

CREATE DATABASE
```

Подключимся к БД:
```bash
\c replica_overview

You are now connected to database "replica_overview" as user "postgres".
```

Созададим таблицу:
```bash
CREATE TABLE test(id integer PRIMARY KEY, description text);

CREATE TABLE
```

Создадим строку:
```bash
INSERT INTO test VALUES (1, 'one');

INSERT 0 1
```

Подключимся из реплики в БД и проверим таблицу:
```bash
SELECT * FROM test;

 id | description 
----+-------------
  1 | one
(1 row)

```

Изменения в реплике не допускаются:
```bash
INSERT INTO test VALUES (2, 'two');

ERROR:  cannot execute INSERT in a read-only transaction
```

По умолчанию реплика работает в режиме горячего резерва - разрешены клиентские подключения только для чтения данных.

Допускаются:
- запросы на чтение данных (`SELECT`, `COPY TO`, курсоры)
- установка параметров сервера (`SET`, `RESET`)
- управления транзакциями (`BEGIN`, `COMMIT`, `ROLLBACK`)
- создание резервной копии (`pg_basebackup`)

Не допускаются:
- любые изменения (`INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `nextval`, ...)
- блокировки, предполагающие изменения (`SELECT FOR UPDATE`, ...)
- команды DDL (`CREATE`, `DROP`, ...) в том числе создание временных таблиц
- команды сопровождения (`VACUUM`, `ANALYZE`, `REINDEX`, ...)
- управление доступом (`GRANT`, `REVOKE`, ...)
- не срабатывают триггеры и рекомендательные блокировки
