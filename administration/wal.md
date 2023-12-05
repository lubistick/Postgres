# Буферный кэш и журнал предварительной записи

## Буферный кэш

В общей памяти экземпляра отводится какая-то часть памяти, которая представляет собой массив буферов.
Каждый буфер хранит в себе одну страницу данных плюс дополнительную информацию - откуда эта страница была прочитана, в каком она состоянии и т.д.
Страница данных, как правило, `8 КБ`, указывается при сборке Postgres.

Когда процессу для выполнения запроса требуются данные, он ищет их сначала в буферном кэше.
Если там их нет, то процесс обращается к ОС.
При этом ОС ищет данные сначала у себя в кэше.
Если не обнаруживает - читает фрагмент данных с диска, кладет их себе в кэш.
Затем Postgres забирает нужную страницу и кладет к себе в буферный кэш.

Посмотрим размер буферного кэша:
```sql
SHOW shared_buffers;

 shared_buffers 
----------------
 128MB
(1 row)
```

Значение по умолчанию слишком мало - `128 МБ`.
В любой реальной системе его требуется увеличить сразу после установки сервера (изменение требует перезапуска).
Условно стандартная рекомендация - начинать примерно с `1/4 объема ОЗУ`, которая доступна экземпляру.


### Вытеснение редко используемых страниц

Когда в кэше заканчиваются свободные буферы, происходит вытеснение страниц с помощью алгоритма `LRU` (`Least Recently Used`).
Т.е. в кеше остаются те страницы, к которым обращаются более часто, остальные вытесняются.

Если процесс изменил данные на странице, соответствующий буфер считается "грязным".
И прежде чем страница будет вытеснена, она будет записана обратно на диск.
Процесс фоновой записи `bgwriter` заранее скидывает на диск "грязные" страницы, которые в скором времени будут вытеснены.


### Влияние буферного кэша на выполнение запросов

Создадим таблицу:
```sql
CREATE TABLE t(n integer);

CREATE TABLE
```

Заполним ее некоторым количеством строк:
```sql
INSERT INTO t SELECT id FROM generate_series(1,100000) AS id;

INSERT 0 100000
```

Запустим очистку и анализ этой таблицы, чтобы Postgres понимал, какого размера таблица:
```sql
VACUUM ANALYZE t;

VACUUM
```

Теперь перезапустим сервер, чтобы содержимое буферного кэша сбросилось:
```bash
docker stop postgres-demo

postgres-demo
```
```bash
docker start postgres-demo

postgres-demo
```

Сравним, что происходит при первом и при втором выполнении одного и того же запроса.
Воспользуемся командой `EXPLAIN ANALYZE`, которая выполняет запрос и выводит не только план выполнения, но и дополнительную информацию:
```sql
EXPLAIN (analyze, buffers, costs off, timing off)
SELECT * FROM t;

                 QUERY PLAN
--------------------------------------------
 Seq Scan on t (actual rows=100000 loops=1) 
   Buffers: shared read=443
 Planning:
   Buffers: shared hit=12 read=8 dirtied=1
 Planning Time: 2.668 ms
 Execution Time: 70.465 ms
(6 rows)
```

`Buffers: shared read` - количество буферов, в которые пришлось прочитать страницы с диска (`443 шт`).

Выполним запрос еще раз:
```sql
EXPLAIN (analyze, buffers, costs off, timing off)
SELECT * FROM t;

                 QUERY PLAN                 
--------------------------------------------
 Seq Scan on t (actual rows=100000 loops=1)
   Buffers: shared hit=443
 Planning Time: 0.024 ms
 Execution Time: 4.222 ms
(4 rows)
```

`Buffers: shared hit` - количество буферов, в которых нашлись нужные для запроса страницы (`443 шт`).

Обратите внимание, что во второй раз уменьшилось время выполнения запроса и время его планирования.


## Журнал предварительной записи

Наличие буферного кэша увеличивает производительность, но уменьшает надежность.
В случае сбоя в СУБД содержимое буферного кэша потеряется.
Для обеспечения надежности Postgres использует журнал предварительной записи `WAL` (`Write Ahead Log`).
Это поток информации о выполняемых действиях.

Перед выполнением любой операции со страницей данных, формируется журнальная запись, содержащая минимально необходимую информацию для того, чтобы операцию можно было выполнить повторно.
Журнальная запись попадает на диск (в энергонезависимую память) раньше, чем измененная страница.
Журнал защищает все объекты, работа с которыми ведется в ОЗУ: таблицы, индексы, статусы транзакций и др.

Физически журнал хранится в файлах по `16 МБ` в отдельном каталоге `PGDATA/pg_wal`.
На них можно взглянуть не только в файловой системе, но и с помощью функции:
```sql
SELECT * FROM pg_ls_waldir() ORDER BY name LIMIT 5;

           name           |   size   |      modification      
--------------------------+----------+------------------------
 0000000100000000000000A5 | 16777216 | 2023-11-28 17:34:39+00 
 0000000100000000000000A6 | 16777216 | 2023-11-12 11:53:33+00 
 0000000100000000000000A7 | 16777216 | 2023-11-12 11:53:36+00 
 0000000100000000000000A8 | 16777216 | 2023-11-12 11:55:47+00 
 0000000100000000000000A9 | 16777216 | 2023-11-12 11:53:42+00 
(5 rows)
```

Логически журнал можно представить в виде непрерывного потока записей.
Каждая запись имеет номер, называемый `LSN` (`Log Sequence Number`).
Это `64-разрядное` число - смещение записи в байтах относительно начала журнала.

Посмотрим текущую позицию:

```sql
SELECT pg_current_wal_lsn();

 pg_current_wal_lsn 
--------------------
 0/A54BF718
(1 row)
```
Позиция записывается как два `32-разрядных` числа через косую черту.

Запомним позицию в переменной "pos1":
```sql
SELECT pg_current_wal_lsn() AS pos1 \gset
```

Теперь выполним какие-нибудь операции и посмотрим, как изменилась позиция:
```sql
CREATE TABLE t(n integer);

CREATE TABLE
```

```sql
INSERT INTO t SELECT gen.id FROM generate_series(1,1000) AS gen(id);

INSERT 0 1000
```

```sql
SELECT pg_current_wal_lsn();

 pg_current_wal_lsn 
--------------------
 0/A54E9F00
(1 row)
```

Позиция изменилась. Запомним ее в переменной "pos2":
```sql
SELECT pg_current_wal_lsn() AS pos2 \gset
```

Найдем разницу между этими позициями в байтах:
```sql
SELECT :'pos2'::pg_lsn - :'pos1'::pg_lsn;

 ?column? 
----------
   221024
(1 row)
```

Примерно `220 КБ` журнала ушло на эти действия.


## Контрольная точка

Процесс `checkpointer` периодически выполняет контрольную точку - принудительно сбрасывает все "грязные" буферы на диск.

Когда Postgres запускается после сбоя, он читает журнал предварительной записи, начиная с последней контрольной точки, и последовательно проигрывает каждую запись, если соответствующее изменение не попало на диск.

Postgres автоматически удаляет журнальные файлы, не требующиеся для восстановления.


## Производительность

Почему использование журнала эффективнее, чем просто писать изменения на диск?

Журнал - это постоянный поток на запись. Для него обычно используется отдельный физический диск.
В то время как писать страницу на диск - это запись в какое-то произвольное место диска.
Это особенно плохо работает на `HDD` (крутящийся диск), потому что требуется время, чтобы спозиционировать головку на нужную дорожку.

Запись в журнал `WAL` может вестись в разных режимах:
- синхронном (значение по умолчанию)
- асинхронном

За это отвечает конфигурационный параметр `synchronous_commit`, который можно устанавливать на лету:
```sql
SELECT * FROM pg_settings WHERE name = 'synchronous_commit' \gx

-[ RECORD 1 ]---+------------------------------------------------------
name            | synchronous_commit
setting         | on
unit            |
category        | Write-Ahead Log / Settings
short_desc      | Sets the current transaction's synchronization level.
extra_desc      |
context         | user
vartype         | enum
source          | default
min_val         |
max_val         |
enumvals        | {local,remote_write,remote_apply,on,off}
boot_val        | on
reset_val       | on
sourcefile      |
sourceline      |
pending_restart | f
```
### Синхронный режим

Запись на диск происходит сразу при фиксации.
Чтобы журнальная запись не "застряла" в кэше ОС, выполняется вызов `fsync`, это гарантирует, что зафиксированная транзакция попадет на диск.
Но запись - дорогая операция, в течение которой обслуживающий процесс, выполняющий фиксацию данных, должен ждать.

### Асинхронный режим

Записи пишутся фоновым процессом `walwriter` постепенно с небольшой задержкой (примерно 2 раза в секунду).
Надежность уменьшается, зато производительность увеличивается.


## Восстановление при помощи журнала

Режимы остановки сервера:

- `fast` - принудительно завершает сеансы и записывает на диск изменения из ОЗУ
- `smart` - ожидает завершения всех сеансов и записывает на диск изменения из ОЗУ
- `immediate` - принудительно завершает сеансы, НЕ записывает на диск изменения из ОЗУ (при запуске потребуется восстановление)


### Логирование сообщений сервера

Процесс `logger`, отвечающий за логирование сообщений сервера выключен:
```sql
SHOW logging_collector;

 logging_collector 
-------------------
 off
(1 row)
```

Включим логирование:
```sql
ALTER SYSTEM SET logging_collector = on;

ALTER SYSTEM
```

Перезапустим сервер, чтобы применить изменения:
```sql
docker stop postgres-demo

postgres-demo
```

```sql
docker start postgres-demo

postgres-demo
```

Теперь сравним выключение сервера в разных режимах.

### Режим fast

При остановке в режиме `fast` сервер выполняет контрольную точку, чтобы записать все "грязные" страницы на диск.
Для остановки в этом режиме отправим сигнал `SIGINT` главному процессу Postgres с помощью команды `kill`
(идентификатор процесса узнаем из файла `postmaster.pid` в каталоге `PGDATA`):

```bash
kill -INT `head -1 $PGDATA/postmaster.pid`
```

Сервер остановился, запустим его:
```sql
docker start postgres-demo

postgres-demo
```

Посмотрим, куда пишутся сообщения сервера:
```sql
SELECT pg_current_logfile();

          pg_current_logfile
--------------------------------------
 log/postgresql-2023-12-03_134742.log
(1 row)
```

Посмотрим сообщения:
```bash
cat /var/lib/postgresql/data/log/postgresql-2023-12-03_134742.log

2023-12-03 13:47:42.932 UTC [1] LOG:  starting PostgreSQL 14.7 on x86_64-pc-linux-musl, compiled by gcc (Alpine 12.2.1_git20220924-r4) 12.2.1 20220924, 64-bit
2023-12-03 13:47:42.932 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2023-12-03 13:47:42.932 UTC [1] LOG:  listening on IPv6 address "::", port 5432
2023-12-03 13:47:42.938 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2023-12-03 13:47:42.945 UTC [23] LOG:  database system was shut down at 2023-12-03 13:46:48 UTC
2023-12-03 13:47:42.950 UTC [1] LOG:  database system is ready to accept connections
```

СУБД готова принимать подключения - `database system is ready to accept connections`.

### Режиме immediate

Режим `immediate` в каком-то смысле равносилен выдергиванию питания сервера.
Сымитируем сбой системы, остановив сервер в этом режиме - отправим сигнал `SIGQUIT` главному процессу Postgres с помощью команды `kill`:

```bash
kill -QUIT `head -1 $PGDATA/postmaster.pid`
```

Сервер остановился, запустим его:
```sql
docker start postgres-demo

postgres-demo
```

Посмотрим, куда пишутся сообщения сервера:
```sql
SELECT pg_current_logfile();

          pg_current_logfile
--------------------------------------
 log/postgresql-2023-12-03_135231.log
(1 row)
```

Посмотрим сообщения:
```bash
cat /var/lib/postgresql/data/log/postgresql-2023-12-03_135231.log

2023-12-03 13:52:31.161 UTC [1] LOG:  starting PostgreSQL 14.7 on x86_64-pc-linux-musl, compiled by gcc (Alpine 12.2.1_git20220924-r4) 12.2.1 20220924, 64-bit
2023-12-03 13:52:31.161 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2023-12-03 13:52:31.161 UTC [1] LOG:  listening on IPv6 address "::", port 5432
2023-12-03 13:52:31.167 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2023-12-03 13:52:31.173 UTC [23] LOG:  database system was interrupted; last known up at 2023-12-03 13:47:42 UTC
2023-12-03 13:52:31.354 UTC [23] LOG:  database system was not properly shut down; automatic recovery in progress
2023-12-03 13:52:31.356 UTC [23] LOG:  redo starts at 0/A5B318B8
2023-12-03 13:52:31.356 UTC [23] LOG:  invalid record length at 0/A5B318F0: wanted 24, got 0
2023-12-03 13:52:31.357 UTC [23] LOG:  redo done at 0/A5B318B8 system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s
2023-12-03 13:52:31.367 UTC [1] LOG:  database system is ready to accept connections
```

СУБД выполнила восстановление `automatic recovery in progress.` После чего готова принимать подключения.

### Режим smart

Для остановки в режиме `smart` нужно отправить сигнал `SIGTERM` главному процессу Postgres с помощью команды `kill`:

```bash
kill -TERM `head -1 $PGDATA/postmaster.pid`
```


## Уровни журнала

Журнал можно применять и для других целей, если добавить в него дополнительную информацию.
Объем данных, которые попадают в журнал регулируется параметром `wal_level`:
```sql
SELECT * FROM pg_settings WHERE name = 'wal_level' \gx

-[ RECORD 1 ]---+--------------------------------------------------
name            | wal_level                                        
setting         | replica                                          
unit            |
category        | Write-Ahead Log / Settings
short_desc      | Sets the level of information written to the WAL.
extra_desc      |
context         | postmaster
vartype         | enum
source          | default
min_val         |
max_val         |
enumvals        | {minimal,replica,logical}
boot_val        | replica
reset_val       | replica
sourcefile      |
sourceline      |
pending_restart | f
```

Значения:
- `minimal` - журнал обеспечивает только восстановление после сбоя
- `replica` - возможность создания резервных копий и репликации (значение по умолчанию)
- `logical` - возможность логической репликации