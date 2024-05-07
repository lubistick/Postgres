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





