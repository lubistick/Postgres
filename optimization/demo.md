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
SELECT * FROM bookings WHERE book_ref = '0824C5';

 book_ref |       book_date        | total_amount
----------+------------------------+--------------
 0824C5   | 2017-07-25 20:36:00+00 |    112400.00
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

"На самом деле" билеты были забронированы 20 дней назад:
```sql
SELECT bookings.now() - book_date FROM bookings WHERE book_ref = '0824C5';

     ?column?
------------------
 20 days 18:24:00
(1 row)
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

Посмотрим, какие билеты включены в бронирование "0824C5"
```sql
SELECT t.* FROM bookings b JOIN tickets t ON t.book_ref = b.book_ref WHERE b.book_ref = '0824C5';

   ticket_no   | book_ref | passenger_id |  passenger_name   |       contact_data
---------------+----------+--------------+-------------------+---------------------------
 0005435126781 | 0824C5   | 7247 393204  | ALEKSANDR MATVEEV | {"phone": "+70095062310"}
 0005435126782 | 0824C5   | 1745 826066  | NINA KRASNOVA     | {"phone": "+70876976071"}
(2 rows)
```

Летят два человека. На каждого оформляется собственный билет с информацией о пассажире.


### Ticket_flights

Перелет соединяет билеты с рейсами.
```sql
\d ticket_flights

                     Table "bookings.ticket_flights"
     Column      |         Type          | Collation | Nullable | Default
-----------------+-----------------------+-----------+----------+---------
 ticket_no       | character(13)         |           | not null |
 flight_id       | integer               |           | not null |
 fare_conditions | character varying(10) |           | not null |
 amount          | numeric(10,2)         |           | not null |
Indexes:
    "ticket_flights_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
Check constraints:
    "ticket_flights_amount_check" CHECK (amount >= 0::numeric)
    "ticket_flights_fare_conditions_check" CHECK (fare_conditions::text = ANY (ARRAY['Economy'::character varying::text, 'Comfort'::character varying::text, 'Business'::character varying::text]))
Foreign-key constraints:
    "ticket_flights_flight_id_fkey" FOREIGN KEY (flight_id) REFERENCES flights(flight_id)
    "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
Referenced by:
    TABLE "boarding_passes" CONSTRAINT "boarding_passes_ticket_no_fkey" FOREIGN KEY (ticket_no, flight_id) REFERENCES ticket_flights(ticket_no, flight_id)
```

Колонки:

- `ticket_no` - номер билета
- `flight_id` - идентификатор рейса
- `fare_conditions` - класс обслуживания
- `amount` - стоимость перелета

По какому маршруту летят пассажиры? Добавим в запрос перелеты:
```sql
SELECT tf.* FROM tickets t JOIN ticket_flights tf ON tf.ticket_no = t.ticket_no WHERE t.ticket_no = '0005435126781';

   ticket_no   | flight_id | fare_conditions |  amount
---------------+-----------+-----------------+----------
 0005435126781 |     22566 | Economy         | 11700.00
 0005435126781 |     71439 | Economy         |  3200.00
 0005435126781 |     74643 | Economy         |  8800.00
 0005435126781 |     94335 | Economy         | 11700.00
 0005435126781 |     95726 | Economy         |  3200.00
 0005435126781 |    206625 | Business        | 26400.00
(6 rows)
```

Смотрим только на один билет - все маршруты в одном бронировании всегда совпадают.


### Flights

Рейс выполняется по расписанию из одного аэропорта в другой.

Естественный ключ - номер рейса и дата отправления, но используется суррогатный ключ.

```sql
\d flights

                                              Table "bookings.flights"
       Column        |           Type           | Collation | Nullable |                  Default
---------------------+--------------------------+-----------+----------+--------------------------------------------
 flight_id           | integer                  |           | not null | nextval('flights_flight_id_seq'::regclass)
 flight_no           | character(6)             |           | not null |
 scheduled_departure | timestamp with time zone |           | not null |
 scheduled_arrival   | timestamp with time zone |           | not null |
 departure_airport   | character(3)             |           | not null |
 arrival_airport     | character(3)             |           | not null |
 status              | character varying(20)    |           | not null |
 aircraft_code       | character(3)             |           | not null |
 actual_departure    | timestamp with time zone |           |          |
 actual_arrival      | timestamp with time zone |           |          |
Indexes:
    "flights_pkey" PRIMARY KEY, btree (flight_id)
    "flights_flight_no_scheduled_departure_key" UNIQUE CONSTRAINT, btree (flight_no, scheduled_departure)
Check constraints:
    "flights_check" CHECK (scheduled_arrival > scheduled_departure)
    "flights_check1" CHECK (actual_arrival IS NULL OR actual_departure IS NOT NULL AND actual_arrival IS NOT NULL AND actual_arrival > actual_departure)
    "flights_status_check" CHECK (status::text = ANY (ARRAY['On Time'::character varying::text, 'Delayed'::character varying::text, 'Departed'::character varying::text, 'Arrived'::character varying::text, 'Scheduled'::character varying::text, 'Cancelled'::character varying::text]))
Foreign-key constraints:
    "flights_aircraft_code_fkey" FOREIGN KEY (aircraft_code) REFERENCES aircrafts_data(aircraft_code)
    "flights_arrival_airport_fkey" FOREIGN KEY (arrival_airport) REFERENCES airports_data(airport_code)
    "flights_departure_airport_fkey" FOREIGN KEY (departure_airport) REFERENCES airports_data(airport_code)
Referenced by:
    TABLE "ticket_flights" CONSTRAINT "ticket_flights_flight_id_fkey" FOREIGN KEY (flight_id) REFERENCES flights(flight_id)
```

Колонки:

- `flight_id` - идентификатор рейса
- `flight_no` - номер рейса
- `scheduled_departure/arrival` - вылет и прилет по расписанию
- `actual_departure/arrival` - фактический вылет и прилет
- `departure/arrival_airport` - аэропорты отправления и прибытия
- `status` - статус рейса
- `aircraft_code` - код самолета

```sql
SELECT f.flight_id, f.scheduled_departure, f.departure_airport dep, f.arrival_airport arr, f.status, f.aircraft_code
FROM tickets t
    JOIN ticket_flights tf ON tf.ticket_no = t.ticket_no
    JOIN flights f ON f.flight_id = tf.flight_id
WHERE t.ticket_no = '0005435126781'
ORDER BY f.scheduled_departure;

 flight_id |  scheduled_departure   | dep | arr |  status   | aircraft_code
-----------+------------------------+-----+-----+-----------+---------------
     22566 | 2017-08-12 08:00:00+00 | VKO | PEE | Arrived   | 773
     95726 | 2017-08-12 12:30:00+00 | PEE | SVX | Arrived   | SU9
     74643 | 2017-08-13 08:30:00+00 | SVX | SGC | Arrived   | SU9
    206625 | 2017-08-15 11:45:00+00 | SGC | SVX | Departed  | SU9
     71439 | 2017-08-16 05:50:00+00 | SVX | PEE | On Time   | SU9
     94335 | 2017-08-16 15:55:00+00 | PEE | VKO | Scheduled | 773
(6 rows)
```

Видим три рейса "туда" и три "обратно".

Туда все рейсы уже совершены (Arrived). В настоящее время пассажир летит "обратно" (Departed).
Следующий рейс будет по расписанию (On time). На последний еще не открыта регистрация (Scheduled).

Посмотрим внимательнее на все столбцы одного из рейсов (22566):
```sql
SELECT * FROM flights WHERE flight_id = 22566 \gx

-[ RECORD 1 ]-------+-----------------------
flight_id           | 22566
flight_no           | PG0412
scheduled_departure | 2017-08-12 08:00:00+00
scheduled_arrival   | 2017-08-12 09:25:00+00
departure_airport   | VKO
arrival_airport     | PEE
status              | Arrived
aircraft_code       | 773
actual_departure    | 2017-08-12 08:01:00+00
actual_arrival      | 2017-08-12 09:25:00+00
```

Реальное время может отличаться от времени по расписанию (обычно не сильно).

Номер `flight_no` одинаков для всех рейсов, следующих по одному маршруту по расписанию:
```sql
SELECT flight_id, flight_no, scheduled_departure
FROM flights WHERE flight_no = 'PG0412' ORDER BY scheduled_departure LIMIT 10;

 flight_id | flight_no |  scheduled_departure
-----------+-----------+------------------------
     22784 | PG0412    | 2016-08-15 08:00:00+00
     22746 | PG0412    | 2016-08-16 08:00:00+00
     22721 | PG0412    | 2016-08-17 08:00:00+00
     22691 | PG0412    | 2016-08-18 08:00:00+00
     22749 | PG0412    | 2016-08-19 08:00:00+00
     22508 | PG0412    | 2016-08-20 08:00:00+00
     22493 | PG0412    | 2016-08-21 08:00:00+00
     22496 | PG0412    | 2016-08-22 08:00:00+00
     22483 | PG0412    | 2016-08-23 08:00:00+00
     22501 | PG0412    | 2016-08-24 08:00:00+00
(10 rows)
```


### Airports

Город не выделен в отдельную таблицу.

Реализация: многоязычное представление над `airports_data`.

```sql
\d airports

                   View "bookings.airports"
    Column    |     Type     | Collation | Nullable | Default
--------------+--------------+-----------+----------+---------
 airport_code | character(3) |           |          |
 airport_name | text         |           |          |
 city         | text         |           |          |
 coordinates  | point        |           |          |
 timezone     | text         |           |          |
```

```sql
\d airports_data

                Table "bookings.airports_data"
    Column    |     Type     | Collation | Nullable | Default
--------------+--------------+-----------+----------+---------
 airport_code | character(3) |           | not null |
 airport_name | jsonb        |           | not null |
 city         | jsonb        |           | not null |
 coordinates  | point        |           | not null |
 timezone     | text         |           | not null |
Indexes:
    "airports_data_pkey" PRIMARY KEY, btree (airport_code)
Referenced by:
    TABLE "flights" CONSTRAINT "flights_arrival_airport_fkey" FOREIGN KEY (arrival_airport) REFERENCES airports_data(airport_code)
    TABLE "flights" CONSTRAINT "flights_departure_airport_fkey" FOREIGN KEY (departure_airport) REFERENCES airports_data(airport_code)
```

Колонки:

- `airport_code` - код аэропорта
- `airport_name` - название аэропорта
- `city` - город
- `coordinates` - координаты аэропорта (долгота и широта)
- `timezone` - часовой пояс

В качестве ключа для аэропортов используется общепринятый трехбуквенный код.

Посмотрим полную информацию об одном аэропорте:
```sql
SELECT * FROM airports WHERE airport_code = 'VKO' \gx

-[ RECORD 1 ]+------------------------------
airport_code | VKO
airport_name | Внуково
city         | Москва
coordinates  | (37.2615013123,55.5914993286)
timezone     | Europe/Moscow
```

Помимо названия и города, хранятся координаты аэропорта и часовой пояс.

Расшифруем сведения о рейсах:
```sql
SELECT f.scheduled_departure,
    dep.airport_code || ' ' || dep.city || ' ' || ' (' || dep.airport_name || ')' departure,
    arr.airport_code || ' ' || arr.city || ' ' || ' (' || arr.airport_name || ')' arrival
FROM tickets t
    JOIN ticket_flights tf ON tf.ticket_no = t.ticket_no
    JOIN flights f ON f.flight_id = tf.flight_id
    JOIN airports dep ON dep.airport_code = f.departure_airport
    JOIN airports arr ON arr.airport_code = f.arrival_airport
WHERE t.ticket_no = '0005435126781'
ORDER BY f.scheduled_departure;

  scheduled_departure   |          departure           |           arrival
------------------------+------------------------------+------------------------------
 2017-08-12 08:00:00+00 | VKO Москва  (Внуково)        | PEE Пермь  (Пермь)
 2017-08-12 12:30:00+00 | PEE Пермь  (Пермь)           | SVX Екатеринбург  (Кольцово)
 2017-08-13 08:30:00+00 | SVX Екатеринбург  (Кольцово) | SGC Сургут  (Сургут)
 2017-08-15 11:45:00+00 | SGC Сургут  (Сургут)         | SVX Екатеринбург  (Кольцово)
 2017-08-16 05:50:00+00 | SVX Екатеринбург  (Кольцово) | PEE Пермь  (Пермь)
 2017-08-16 15:55:00+00 | PEE Пермь  (Пермь)           | VKO Москва  (Внуково)
(6 rows)
```

Чтобы не писать каждый раз подобный запрос, существует представление `flights_v`:
```sql
SELECT * FROM flights_v WHERE flight_id = 22566 \gx

-[ RECORD 1 ]-------------+-----------------------
flight_id                 | 22566
flight_no                 | PG0412
scheduled_departure       | 2017-08-12 08:00:00+00
scheduled_departure_local | 2017-08-12 11:00:00
scheduled_arrival         | 2017-08-12 09:25:00+00
scheduled_arrival_local   | 2017-08-12 14:25:00
scheduled_duration        | 01:25:00
departure_airport         | VKO
departure_airport_name    | Внуково
departure_city            | Москва
arrival_airport           | PEE
arrival_airport_name      | Пермь
arrival_city              | Пермь
status                    | Arrived
aircraft_code             | 773
actual_departure          | 2017-08-12 08:01:00+00
actual_departure_local    | 2017-08-12 11:01:00
actual_arrival            | 2017-08-12 09:25:00+00
actual_arrival_local      | 2017-08-12 14:25:00
actual_duration           | 01:24:00
```

Поскольку в демобазе маршруты не меняются со временем, из таблицы рейсов можно выделить информацию, которая не зависит от конкретной даты вылета. Такая информация собрана в представлении `routes`:
```sql
SELECT * FROM routes WHERE flight_no = 'PG0412' \gx

-[ RECORD 1 ]----------+----------------
flight_no              | PG0412
departure_airport      | VKO
departure_airport_name | Внуково
departure_city         | Москва
arrival_airport        | PEE
arrival_airport_name   | Пермь
arrival_city           | Пермь
aircraft_code          | 773
duration               | 01:25:00
days_of_week           | {1,2,3,4,5,6,7}
```












```sql

```
