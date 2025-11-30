# Резервное копирование

Существует два вида резервирования: логическое и физическое.

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
