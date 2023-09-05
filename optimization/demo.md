# Демонстрационная база данных

Заходим [на сайт авторов](https://postgrespro.ru/education/demodb) и скачиваем самый большой дамп.
Или скачиваем [по этой ссылке](https://edu.postgrespro.ru/demo-big.zip).

## Локальный запуск

Подразумевается что у нас установлен `Docker`.

Распаковываем архив с дампом, например в `/home/lubis/postgres-demo/demo-big-20170815.sql`.

Запускаем контейнер с `Postgres`, обязательно прикручиваем `volume`:

```bash
docker run -e POSTGRES_PASSWORD=postgres -v /home/lubis/postgres-demo:/dump -d --name postgres-demo postgres:14-alpine
```

Заходим внутрь контейнера:

```bash
docker exec -ti postgres-demo sh
```

Накатываем дамп:

```bash
psql -U postgres < /dump/demo-big-20170815.sql
```

Ждем... Готово.


## Обзор таблиц

Заходим через `psql` и подключаемся к базе данных:

```bash
\c demo

You are now connected to database "demo" as user "postgres".
```

### Bookings

Пассажир заранее (за месяц) бронирует билет себе и, возможно, нескольким другим пассажирам:
```sql
\d bookings

                        Table "bookings.bookings"
    Column    |           Type           | Collation | Nullable | Default
--------------+--------------------------+-----------+----------+---------
 book_ref     | character(6)             |           | not null |
 book_date    | timestamp with time zone |           | not null |
 total_amount | numeric(10,2)            |           | not null |
Indexes:
    "bookings_pkey" PRIMARY KEY, btree (book_ref)
Referenced by:
    TABLE "tickets" CONSTRAINT "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
```
Колонки:

- `book_ref` - номер бронирования
- `book_date` - дата бронирования
- `total_amount` - общая стоимость включенных в бронирование билетов

```sql
SELECT * FROM bookings LIMIT 1;

 book_ref |       book_date        | total_amount
----------+------------------------+--------------
 000004   | 2016-08-13 12:40:00+00 |     55800.00
(1 row)
```

Если сравнивать дату с текущей, то бронирование сделано довольно давно:
```sql
SELECT now();

              now
-------------------------------
 2023-09-05 19:02:01.269902+00
(1 row)
```

Но для демо базы "текущим" моментом является другая дата:
```sql
SELECT bookings.now();

          now
------------------------
 2017-08-15 15:00:00+00
(1 row
```

### Tickets

Билет выдается на одного пассажира и может включать несколько перелетов.

Ни идентификатор пассажира, ни имя не являются постоянными. Нельзя однозначно найти все билеты одного и того же пассажира.

```sql
\d tickets

                        Table "bookings.tickets"
     Column     |         Type          | Collation | Nullable | Default
----------------+-----------------------+-----------+----------+---------
 ticket_no      | character(13)         |           | not null |
 book_ref       | character(6)          |           | not null |
 passenger_id   | character varying(20) |           | not null |
 passenger_name | text                  |           | not null |
 contact_data   | jsonb                 |           |          |
Indexes:
    "tickets_pkey" PRIMARY KEY, btree (ticket_no)
Foreign-key constraints:
    "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
Referenced by:
    TABLE "ticket_flights" CONSTRAINT "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
```

Колонки:

- `ticket_no` - номер билета
- `book_ref` - номер бронирования
- `passenger_id` - идентификатор пассажира (номер документа)
- `passenger_name` - имя пассажира
- `contact_data` - контактные данные пассажира
