# Мониторинг

## Настройка

Вначале включим сбор статистики ввода-вывода и выполнения функций:
```sql
ALTER SYSTEM SET track_io_timing = on;

ALTER SYSTEM
```

```sql
ALTER SYSTEM SET track_functions = 'all';

ALTER SYSTEM
```

```sql
SELECT pg_reload_conf();

 pg_reload_conf 
----------------
 t
(1 row)
```

```sql
CREATE DATABASE admin_monitoring;

CREATE DATABASE
```

Смотреть на активности сервера имеет смысл, когда какие-то активности на самом деле есть.
Чтобы сымитировать нагрузку, воспользуемся `pgbench` - штатной утилитой для запуска эталонных тестов.
Сначала утилита создает набор таблиц и заполняет их данными:
```bash
pgbench -i admin_monitoring -U postgres

dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.25 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.12 s, vacuum 0.06 s, primary keys 0.05 s).
```

Сбросим все накопленные ранее статистики:
```sql
SELECT pg_stat_reset();

 pg_stat_reset 
---------------

(1 row)
```

```sql
SELECT pg_stat_reset_shared('bgwriter');

 pg_stat_reset_shared 
----------------------

(1 row)
```

## Статистика

Теперь запускаем тест `TPC-B` на `10` секунд:
```bash
pgbench -T 10 admin_monitoring -U postgres

pgbench (14.7)
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 10 s
number of transactions actually processed: 6607
latency average = 1.513 ms
initial connection time = 2.221 ms
tps = 660.836066 (without initial connection time)
```

```sql
VACUUM pgbench_accounts;

VACUUM
```

Теперь мы можем посмотреть статистику обращения к таблицам в терминах строк:
```sql
SELECT * FROM pg_stat_all_tables WHERE relid='pgbench_accounts'::regclass \gx

-[ RECORD 1 ]-------+-----------------------------
relid               | 106750                      
schemaname          | public                      
relname             | pgbench_accounts            
seq_scan            | 0                           
seq_tup_read        | 0                           
idx_scan            | 13214
idx_tup_fetch       | 13214
n_tup_ins           | 0
n_tup_upd           | 6607
n_tup_del           | 0
n_tup_hot_upd       | 4978
n_live_tup          | 100000
n_dead_tup          | 0
n_mod_since_analyze | 6607
n_ins_since_vacuum  | 0
last_vacuum         | 2024-05-07 14:24:19.48388+00
last_autovacuum     |
last_analyze        |
last_autoanalyze    |
vacuum_count        | 1
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 0
```

И в терминах страниц:
```sql
SELECT * FROM pg_statio_all_tables WHERE relid = 'pgbench_accounts'::regclass \gx

-[ RECORD 1 ]---+-----------------
relid           | 106750
schemaname      | public
relname         | pgbench_accounts
heap_blks_read  | 27
heap_blks_hit   | 38055
idx_blks_read   | 275
idx_blks_hit    | 29765
toast_blks_read |
toast_blks_hit  |
tidx_blks_read  |
tidx_blks_hit   |
```

Существуют аналогичные представления для индексов:
```sql
SELECT * FROM pg_stat_all_indexes WHERE relid = 'pgbench_accounts'::regclass \gx

-[ RECORD 1 ]-+----------------------
relid         | 106750
indexrelid    | 106764
schemaname    | public
relname       | pgbench_accounts
indexrelname  | pgbench_accounts_pkey
idx_scan      | 13214
idx_tup_read  | 14920
idx_tup_fetch | 13214
```

```sql
SELECT * FROM pg_statio_all_indexes WHERE relid = 'pgbench_accounts'::regclass \gx

-[ RECORD 1 ]-+----------------------
relid         | 106750
indexrelid    | 106764
schemaname    | public
relname       | pgbench_accounts
indexrelname  | pgbench_accounts_pkey
idx_blks_read | 275
idx_blks_hit  | 29765
```

А также варианты для пользовательских и системных объектов (all, user, sys), для статистики текущей транзакции (pg_stat_xact*) и др.

Можно посмотреть глобальную статистику по всей БД:
```sql
SELECT * FROM pg_stat_database WHERE datname = 'admin_monitoring' \gx

-[ RECORD 1 ]------------+------------------------------
datid                    | 106743
datname                  | admin_monitoring
numbackends              | 1
xact_commit              | 6731
xact_rollback            | 1
blks_read                | 434
blks_hit                 | 120588
tup_returned             | 150492
tup_fetched              | 15283
tup_inserted             | 6613
tup_updated              | 19829
tup_deleted              | 0
conflicts                | 0
temp_files               | 0
temp_bytes               | 0
deadlocks                | 0
checksum_failures        |
checksum_last_failure    | 
blk_read_time            | 3.062
blk_write_time           | 0
session_time             | 16695247.246
active_time              | 7321.928
idle_in_transaction_time | 2400.179
sessions                 | 3
sessions_abandoned       | 0
sessions_fatal           | 0
sessions_killed          | 0
stats_reset              | 2024-05-07 14:18:17.738689+00
```

Отдельно доступна статистика по процессам фоновой записи и контрольной точки, ввиду ее важности для мониторинга экземпляра:
```sql
CHECKPOINT;

CHECKPOINT
```

```sql
SELECT * FROM pg_stat_bgwriter \gx

-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 10
checkpoints_req       | 1
checkpoint_write_time | 370413
checkpoint_sync_time  | 32
buffers_checkpoint    | 3750
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 79
buffers_backend_fsync | 0
buffers_alloc         | 432
stats_reset           | 2024-05-07 14:19:12.208774+00
```
- `buffers_clean` - количество страниц, записанных фоновой записью
- `buffers_checkpoint` - количество страниц, записанных контрольной точкой
- `buffers_bachend` - количество страниц, записанных серверными процессами


## Текущие активности

Воспроизведем сценарий, в котором один процесс блокирует выполнение другого,
и попробуем разобраться в ситуации с помощью системных представлений.
Создадим таблицу с одной строкой:
```sql
CREATE TABLE t(n integer);

CREATE TABLE
```

```sql
INSERT INTO t VALUES(42);

INSERT 0 1
```

Запустим два сеанса, один из которых изменяет таблицу и ничего не делает:
```sql
BEGIN;

BEGIN
```

```sql
UPDATE t SET n = n + 1;

UPDATE 1
```

А второй пытается изменить ту же строку и блокируется:
```bash
psql -U postgres -d admin_monitoring
```

```sql
UPDATE t SET n = n + 2;
```

Посмотрим информацию об обслуживающих процессах:
```sql
SELECT pid, query, state, wait_event, wait_event_type, pg_blocking_pids(pid) FROM pg_stat_activity WHERE backend_type = 'client backend' \gx
-[ RECORD 1 ]----+------------------------------------------------------------------------------------------------------------------------------------------
pid              | 355                                                                                                                                      
query            | UPDATE t SET n = n + 2;                                                                                                                  
state            | active                                                                                                                                   
wait_event       | transactionid                                                                                                                            
wait_event_type  | Lock
pg_blocking_pids | {114}
-[ RECORD 2 ]----+------------------------------------------------------------------------------------------------------------------------------------------
pid              | 114
query            | UPDATE t SET n = n + 1;
state            | idle in transaction
wait_event       | ClientRead
wait_event_type  | Client
pg_blocking_pids | {}
-[ RECORD 3 ]----+------------------------------------------------------------------------------------------------------------------------------------------
pid              | 370
query            | SELECT pid, query, state, wait_event, wait_event_type, pg_blocking_pids(pid) FROM pg_stat_activity WHERE backend_type = 'client backend'
state            | active
wait_event       |
wait_event_type  |
pg_blocking_pids | {}
```

Состояние `idle in transaction` означает, что сеанс начал транзакцию, но в настоящее время ничего не делает, а транзакция осталась незавершенной.
Это может стать проблемой, если ситуация возникает систематически, например, из-за некорректной реализации приложения или из-за ошибок в драйвере, поскольку открытый сеанс расходует оперативную память.

Покажем как завершить сеанс вручную.
Запомним номер заблокированного процесса:
```sql
SELECT pid AS blocked_pid FROM pg_stat_activity WHERE backend_type = 'client backend' AND cardinality(pg_blocking_pids(pid)) > 0 \gset
```

В `Postgres` версии ниже `9.6` нет функции `pg_blocking_pids(<ID_процесса>)`, но блокирующий процесс можно вычислить, используя запросы к таблице блокировок.
Запрос покажет две строки: одна транзакция получила блокировку (`granted`), другая - нет и ожидает:
```sql
SELECT locktype, transactionid, pid, mode, granted FROM pg_locks WHERE transactionid IN (SELECT transactionid FROM pg_locks WHERE pid = :blocked_pid AND NOT granted);

   locktype    | transactionid | pid |     mode      | granted 
---------------+---------------+-----+---------------+---------
 transactionid |          7677 | 114 | ExclusiveLock | t
 transactionid |          7677 | 355 | ShareLock     | f
(2 rows)
```

В общем случае нужно аккуратно учитывать тип блокировки.

Выполнение запроса можно прервать функцией `pg_cancel_backend`.
В нашем случае транзакция простаивает, так что просто прерываем сеанс, вызвав `pg_terminate_backend`:
```sql
SELECT pg_terminate_backend(b.pid) FROM unnest(pg_blocking_pids(:blocked_pid)) AS b(pid);

 pg_terminate_backend 
----------------------
 t
(1 row)
```

Функция `unnest` нужна, поскольку `pg_blocking_pids` возвращает массив идентификаторов процессов, блокирующих искомый серверный процесс.
В нашем примере блокирующий процесс один, но в общем случае их может быть несколько.

Проверим состояние серверных процессов:
```sql
SELECT pid, query, state, wait_event, wait_event_type, pg_blocking_pids(pid) FROM pg_stat_activity WHERE backend_type = 'client backend' \gx

-[ RECORD 1 ]----+------------------------------------------------------------------------------------------------------------------------------------------
pid              | 355
query            | UPDATE t SET n = n + 2;
state            | idle
wait_event       | ClientRead
wait_event_type  | Client
pg_blocking_pids | {}
-[ RECORD 2 ]----+------------------------------------------------------------------------------------------------------------------------------------------
pid              | 370
query            | SELECT pid, query, state, wait_event, wait_event_type, pg_blocking_pids(pid) FROM pg_stat_activity WHERE backend_type = 'client backend'
state            | active
wait_event       |
wait_event_type  |
pg_blocking_pids | {}
```

Осталось только два, причем заблокированный успешно завершил транзакцию.
Если в первом терминале попробовать закоммитить изменения:
```sql
COMMIT;

FATAL:  terminating connection due to administrator command
server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
The connection to the server was lost. Attempting reset: Succeeded.
```

Если постмастер заметит что какой то процесс пропал неожиданно (тайминг 52 минуты) ...дописать про kill -9


Начиная с версии `10` представление `pg_stat_activity` показывает информацию не только про обслуживающие процессы, но и про служебные фоновые процессы экземпляра:
```sql
SELECT pid, backend_type, backend_start, state FROM pg_stat_activity;

 pid |         backend_type         |         backend_start         | state  
-----+------------------------------+-------------------------------+--------
  29 | logical replication launcher | 2024-05-07 14:01:48.324964+00 |
  27 | autovacuum launcher          | 2024-05-07 14:01:48.325743+00 |
 355 | client backend               | 2024-05-07 19:42:35.568334+00 | idle
 370 | client backend               | 2024-05-07 19:45:05.451273+00 | active
  25 | background writer            | 2024-05-07 14:01:48.324473+00 |
  24 | checkpointer                 | 2024-05-07 14:01:48.326223+00 |
  26 | walwriter                    | 2024-05-07 14:01:48.324803+00 |
(7 rows)
```


## Анализ журнала сервера

```sql
select pg_current_logfile();

          pg_current_logfile          
--------------------------------------
 log/postgresql-2024-05-07_140148.log
(1 row)
```

```bash
grep FATAL /var/lib/postgresql/data/log/postgresql-2024-05-07_140148.log | tail -n 1

2024-05-07 20:21:56.016 UTC [114] FATAL:  terminating connection due to administrator command
```

Сообщение `terminating connection` вызвано тем, что мы завершали блокирующий процесс.


Обычное применение журнала - анализ наиболее продолжительных запросов.
Добавим к строкам журнала номер процесса и включим вывод команд и времени их выполнения:
```sql
ALTER SYSTEM SET log_min_duration_statement = 0;

ALTER SYSTEM
```
Это означает, что абсолютно все команды будут попадать в журнал.

```sql
ALTER SYSTEM SET log_line_prefix = '(pid=%p) ';

ALTER SYSTEM
```

```sql
SELECT pg_reload_conf();

 pg_reload_conf 
----------------
 t
(1 row)
```

Теперь выполним какую-нибудь команду:
```sql
SELECT sum(random()) FROM generate_series(1,1000000);

        sum        
-------------------
 499925.5793404646
(1 row)
```

И посмотрим на последнюю строку в журнале:
```bash
tail -n 1 /var/lib/postgresql/data/log/postgresql-2024-05-07_140148.log

(pid=370) LOG:  duration: 252.063 ms  statement: SELECT sum(random()) FROM generate_series(1,1000000);
```

Для полноценного мониторинга требуется внешняя система.

---------------------------------------------------------------
ПРАКТИКА


## Статистика обращений к таблице

Создадим таблицу так, чтобы ее не трогал `autovacuum`:
```sql
CREATE TABLE tt(n numeric) WITH (autovacuum_enabled = off);

CREATE TABLE
```

Вставим `1 000` строк:
```sql
INSERT INTO tt SELECT 1 FROM generate_series(1, 1000);

INSERT 0 1000
```

Удалим все строки из таблицы:
```sql
DELETE FROM tt;

DELETE 1000
```

Проверяем статистику обращений:
```sql
SELECT * FROM pg_stat_all_tables WHERE relid = 'tt'::regclass \gx

-[ RECORD 1 ]-------+-------
relid               | 106775
schemaname          | public
relname             | tt
seq_scan            | 1
seq_tup_read        | 1000
idx_scan            |
idx_tup_fetch       |
n_tup_ins           | 1000
n_tup_upd           | 0
n_tup_del           | 1000
n_tup_hot_upd       | 0
n_live_tup          | 0
n_dead_tup          | 1000
n_mod_since_analyze | 2000
n_ins_since_vacuum  | 1000
last_vacuum         |
last_autovacuum     |
last_analyze        |
last_autoanalyze    |
vacuum_count        | 0
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 0
```

- `n_tup_ins` - мы вставили `1000` строк
- `n_tup_del` - удалили `1000` строк
- `n_live_tup` имеет значение `0` - после удаления не осталось активных версий строк
- `n_dead_tup` имеет значение `1000` - все версии строк не актуальны на текущий момент

Выполним очистку:
```sql
VACUUM;

VACUUM
```

```sql
SELECT * FROM pg_stat_all_tables WHERE relid = 'tt'::regclass \gx

-[ RECORD 1 ]-------+------------------------------
relid               | 106775
schemaname          | public
relname             | tt
seq_scan            | 1
seq_tup_read        | 1000
idx_scan            |
idx_tup_fetch       |
n_tup_ins           | 1000
n_tup_upd           | 0
n_tup_del           | 1000
n_tup_hot_upd       | 0
n_live_tup          | 0
n_dead_tup          | 0
n_mod_since_analyze | 2000
n_ins_since_vacuum  | 0
last_vacuum         | 2024-05-08 13:56:32.252112+00
last_autovacuum     |
last_analyze        |
last_autoanalyze    |
vacuum_count        | 1
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 0
```

- `n_dead_tup` имеет значение `0` - очистка убрала неактуальные версии строк
- `vacuum_count` имеет значение `1` - очистка обрабатывала таблицу `1` раз

