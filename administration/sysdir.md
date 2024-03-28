# Системный каталог

В схеме `pg_catalog` располагаются таблицы и представления, которые описывают все объекты кластера баз данных.

Стандартом SQL регламентировано само понятие системного каталога и каким образом с ним нужно работать.
Postgres поддерживает стандарт, поэтому к системному каталогу можно обратиться и через схему `information_schema.`

Схема `pg_catalog` есть в каждой БД кластера.


## Некоторые объекты системного каталога

Создадим БД и текстовые объекты:

```sql
CREATE DATABASE data_catalog;

CREATE DATABASE
```

```sql
\c data_catalog

You are now connected to database "data_catalog" as user "postgres".
```

```sql
CREATE TABLE employees(
  id serial PRIMARY KEY,
  name text,
  manager integer
);

CREATE TABLE
```

```sql
CREATE VIEW top_manager AS
SELECT * FROM employees WHERE manager IS NULL;

CREATE VIEW
```

Некоторые таблицы системного каталога нам уже знакомы из предыдущей темы. Это базы данных:

```sql
SELECT * FROM pg_database WHERE datname = 'data_catalog' \gx

-[ RECORD 1 ]-+-------------
oid           | 65720
datname       | data_catalog
datdba        | 10
encoding      | 6
datcollate    | en_US.utf8
datctype      | en_US.utf8
datistemplate | f
datallowconn  | t
datconnlimit  | -1
datlastsysoid | 13776
datfrozenxid  | 727
datminmxid    | 1
dattablespace | 1663
datacl        |
```






---

Что такое системный каталог и как к нему обращаться

Объекты системного каталога и их расположение

Правила именования объектов

Специальные типы данных

---


SQL-доступ

просмотр: SELECT
изменение: CREATE, ALTER, DROP

Доступ в psql

специальные команды для удобного просмотра

---


OID - тип для идентификатора объекта

первичные и внешние ключи в таблицах системного каталога

скрытый столбец, в запросах надо указывать явно


Reg-типы

псевдонимы OID для некоторых таблиц системного каталога (regclass для pg_class и т.п.)

привидение текстового имени объекта к типу OID и обратно





остановился на 12:22
