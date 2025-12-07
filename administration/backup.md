# Резервное копирование

Существует два вида резервирования: логическое и физическое.


## Логическое копирование

Логическая резервная копия - это набор команд SQL, восстанавливающих кластер, базу данных или отдельный объект с нуля. Такая копия представляет собой обычный текстовый файл.

Создадим БД и таблицу в ней:
```sql
CREATE DATABASE backup_overview;

CREATE DATABASE
```

```sql
CREATE TABLE t(id numeric, s text);

CREATE TABLE
```

```sql
INSERT INTO t VALUES (1, 'Hello'), (2, ''), (3, NULL);

INSERT 0 3
```

```sql
SELECT * FROM t;

 id |   s
----+-------
  1 | Hello
  2 |
  3 |
(3 rows)
```

### Логическая копия таблицы

Команда `COPY` позволяет записать таблицу либо в файл, либо в поток вывода, либо на вход произвольной команде:

```sql
COPY t TO STDOUT;

1       Hello
2
3       \N
```

Вот как выглядит таблица в выводе команды `COPY`.
Обратим внимание на то, что пустая строка и `NULL` - разные значения, хотя, выполняя запрос, этого и не заметно.

Аналогично можно вводить данные:
```sql
TRUNCATE TABLE t;

TRUNCATE TABLE
```

```sql
COPY t FROM STDIN;

Enter data to be copied followed by a newline.
End with a backslash and a period on a line by itself, or an EOF signal.

>> 1    Hi there!
>> 2
>> 3    \N
>> \.
COPY 3
```

Значения колонок разделяются табуляцией.
Если новая строка начинается с последовательности символов `\.`, ввод данных прекращается.

Поскольку команда `COPY` серверная, то файлы создаются на сервере. В `psql` есть полный аналог этой команды `\COPY`. Это позволяет нам через `psql` удаленно подключиться к какой-то базе и выгрузить какую-то таблицу или наоборот загрузить.


### Логическая копия всей БД

Для создания полноценной резервной копии базы данных существует утилита `pg_dump`.

```sql
pg_dump -U postgres -d backup_overview --create
--
-- PostgreSQL database dump
--

\restrict vC3Cwxd2inppOHajB1Ts0h1rHeg0gVCPoyzLUgc0k6euzsqeUS6fdx1O0z1J69D

-- Dumped from database version 18.1
-- Dumped by pg_dump version 18.1

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET transaction_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

--
-- Name: backup_overview; Type: DATABASE; Schema: -; Owner: postgres
--

CREATE DATABASE backup_overview WITH TEMPLATE = template0 ENCODING = 'UTF8' LOCALE_PROVIDER = libc LOCALE = 'en_US.utf8';


ALTER DATABASE backup_overview OWNER TO postgres;

\unrestrict vC3Cwxd2inppOHajB1Ts0h1rHeg0gVCPoyzLUgc0k6euzsqeUS6fdx1O0z1J69D
\connect backup_overview
\restrict vC3Cwxd2inppOHajB1Ts0h1rHeg0gVCPoyzLUgc0k6euzsqeUS6fdx1O0z1J69D

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET transaction_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

SET default_tablespace = '';

SET default_table_access_method = heap;

--
-- Name: t; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.t (
    id numeric,
    s text
);


ALTER TABLE public.t OWNER TO postgres;

--
-- Data for Name: t; Type: TABLE DATA; Schema: public; Owner: postgres
--

COPY public.t (id, s) FROM stdin;
1       Hello
2
3       \N
\.


--
-- PostgreSQL database dump complete
--

\unrestrict vC3Cwxd2inppOHajB1Ts0h1rHeg0gVCPoyzLUgc0k6euzsqeUS6fdx1O0z1J69D
```

Результатом работы команды в данном случае является SQL-скрипт, содержащий команды, создающие выбранные объекты в БД. Опция `--create` добавит команду создания БД в скрипт.
Утилита `pg_dump` выгружает строки таблиц в формате команды `COPY` (можно изменить на формат `INSERT`), т.к. команда `COPY` при массовой вставке работает существенно быстрее, чем команда `INSERT`.


### Восстановление

Базу данных для восстановления надо создавать из шаблона `template0`, т.к. все изменения, сделанные в `template1` также попадут в резервную копию.
Кроме того, заранее должны быть созданы необходимые роли и табличные пространства, поскольку эти объекты относятся ко вмему кластеру.
После восстановления базы имеет смысл выполнить команду `ANALYZE`, которая соберет статистику.
Создадим вторую БД:

```sql
CREATE DATABASE backup_overview_2;

CREATE DATABASE
```

Можно выгружать только отдельные объекты. Напрмер, скопируем таблицу `t` в другую базу данных.
Чтобы восстановить объекты из SQL-скрипта, достаточно прогнать его через `psql`.
Подадим вывод команды `pg_dump` на вход команде `psql` для восстановления из дампа:

```sql
pg_dump -U postgres -d backup_overview --table=t | psql -U postgres -d backup_overview_2

SET
SET
SET
SET
SET
SET
 set_config 
------------
 
(1 row)

SET
SET
SET
SET
SET
SET
CREATE TABLE
ALTER TABLE
COPY 3
```

Подключимся ко второй базе данных:
```sql
\c backup_overview_2 

You are now connected to database "backup_overview_2" as user "postgres".
```

И проверим, что данные накатились:
```sql
SELECT * FROM t;

 id |   s   
----+-------
  1 | Hello
  2 | 
  3 | 
(3 rows)
```

## Физическое копирование

Физическая резервная копия представляет собой копию файловой системы с Postgres кластером.
Нельзя восстановить отдельную БД, а только кластер целиком на определенный момент времени.
Есть возможность создать копию "на лету", не выключая сервер.

В корне проекта заранее подготовлен файл `docker-compose.yaml` с двумя кластерами.
Будем делать копию кластера "postgres-main" в "postgres-replica".
Кластер "postgres-replica" должен быть остановлен, для этого раскомментируем `command: ["sleep", "infinity"]`.

Запустим кластеры:

```bash
docker compose up -d
[+] Running 3/3
 ✔ Network postgres_default       Created     0.0s
 ✔ Container postgres-main        Started     0.1s
 ✔ Container postgres-replica     Started     0.1s
```

Зайдем в кластер мэйн:

```bash
docker exec -ti postgres-main sh

/ #
```

В другом окне терминала заходим в реплику:

```bash
docker exec -ti postgres-replica sh

/ #
```

Для создания горячей резервной копии существует утилита `pg_basebackup`.
Вначале утилита выполняет контрольную точку, затем копируется файловая система кластера.
А также все файлы WAL, сгенерированные сервером за время от контрольной точки до окончания копирования файлов.

Из консоли "postgres-replica" сделаем бэкап кластера "postgres-main":
```bash
pg_basebackup -U postgres -h postgres-main --pgdata /home/basebackup

pg_basebackup: error: connection to server at "postgres-main" (172.22.0.3), port 5432 failed: FATAL:  no pg_hba.conf entry for replication connection from host "172.22.0.2", user "postgres", no encryption
```

Мы получили ошибку. Для подключения необходим ряд настроек на том кластере, который мы будем копировать (т.е. нужно настроить мэйн).


### Настройка

1. Утилита `pg_basebackup` подключается к серверу по специальному протоколу репликации.
Параметр `wal_level`, определяющий количество информации в журнале, должен быть установлен в значение `replica`:
```sql
SELECT name, setting FROM pg_settings WHERE name IN ('wal_level');

   name    | setting 
-----------+---------
 wal_level | replica
(1 row)
```

2. Параметр `max_wal_senders` должен быть утановлен в достаточно большое значение.
Этот параметр ограничивает число одновременно работающих процессов `wal_sender`, обслуживающих подключение по протоколу репликации:

```sql
SELECT name, setting FROM pg_settings WHERE name IN ('max_wal_senders');

      name       | setting 
-----------------+---------
 max_wal_senders | 10
(1 row)
```

3. Роль должна обладать атрибутом `REPLICATION` или быть суперпользователем.
Пользовать "postgres" является суперпользователем.

4. Роли должно быть выдано разрешение в конфигурационном файле `pg_hba.conf`.
Разрешение на локальное подключение по протоколу репликации прописано по умолчанию:

```sql
select type, database, user_name, address, auth_method
from pg_hba_file_rules()
where 'replication' = ANY(database);

 type  |   database    | user_name |  address  | auth_method 
-------+---------------+-----------+-----------+-------------
 local | {replication} | {all}     |           | trust
 host  | {replication} | {all}     | 127.0.0.1 | trust
 host  | {replication} | {all}     | ::1       | trust
(3 rows)
```

Итак, ошибка возникла, потому что в файле `pg_hba.conf` отсетствует правило для репликационных подключений от хоста "172.22.0.2" с пользователем "postgres".

Поправим конфиг через vim:

```bash
vi $PGDATA/pg_hba.conf
```

Добавим строку `host replication postgres 0.0.0.0/0 trust`, разрешающую подключение с любого айпи (на реальном кластере так делать не надо).
Получилось так:
```bash
tail $PGDATA/pg_hba.conf

# IPv6 local connections:
host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust
host    replication     postgres        0.0.0.0/0               trust
```

Как альтернатива, мы могли бы указать такой айпи: `172.22.0.2/16`.

Перезагрузим конфигурацию сервера:

```sql
SELECT pg_reload_conf();

 pg_reload_conf 
----------------
 t
(1 row)
```

Теперь попробуем сделать копию из терминала с репликой:

```bash
pg_basebackup -U postgres -h postgres-main --pgdata /home/basebackup
```

Ошибок нет. Проверим папку с копией:

```bash
ls -la /home/basebackup

total 268
drwx------   19 root     root          4096 Dec  7 11:11 .
drwxr-xr-x    1 root     root          4096 Dec  7 11:11 ..
-rw-------    1 root     root             3 Dec  7 11:11 PG_VERSION
-rw-------    1 root     root           225 Dec  7 11:11 backup_label
-rw-------    1 root     root        137645 Dec  7 11:11 backup_manifest
drwx------    5 root     root          4096 Dec  7 11:11 base
drwx------    2 root     root          4096 Dec  7 11:11 global
drwx------    2 root     root          4096 Dec  7 11:11 pg_commit_ts
drwx------    2 root     root          4096 Dec  7 11:11 pg_dynshmem
-rw-------    1 root     root          5823 Dec  7 11:11 pg_hba.conf
-rw-------    1 root     root          2681 Dec  7 11:11 pg_ident.conf
drwx------    4 root     root          4096 Dec  7 11:11 pg_logical
drwx------    4 root     root          4096 Dec  7 11:11 pg_multixact
drwx------    2 root     root          4096 Dec  7 11:11 pg_notify
drwx------    2 root     root          4096 Dec  7 11:11 pg_replslot
drwx------    2 root     root          4096 Dec  7 11:11 pg_serial
drwx------    2 root     root          4096 Dec  7 11:11 pg_snapshots
drwx------    2 root     root          4096 Dec  7 11:11 pg_stat
drwx------    2 root     root          4096 Dec  7 11:11 pg_stat_tmp
drwx------    2 root     root          4096 Dec  7 11:11 pg_subtrans
drwx------    2 root     root          4096 Dec  7 11:11 pg_tblspc
drwx------    2 root     root          4096 Dec  7 11:11 pg_twophase
drwx------    4 root     root          4096 Dec  7 11:11 pg_wal
drwx------    2 root     root          4096 Dec  7 11:11 pg_xact
-rw-------    1 root     root            88 Dec  7 11:11 postgresql.auto.conf
-rw-------    1 root     root         32307 Dec  7 11:11 postgresql.conf
```

Получилось.


todo дописать

Для восстановления достаточно развернуть резервную копию и запустить сервер.
При необходимости он выполнит восстановление согасованности с помощью файлов WAL и будет готов к работе.



