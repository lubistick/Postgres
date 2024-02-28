# Сканирование по битовой карте

Будем рассматривать таблицу бронирований bookings.

Создадим на ней два дополнительных индекса:

```sql
CREATE INDEX ON bookings(book_date);

CREATE INDEX
```

```sql
CREATE INDEX ON bookings(total_amount);

CREATE INDEX
```

Посмотрим, какой метод доступа будет выбран для поиска диапазона:
```sql
EXPLAIN SELECT * FROM bookings WHERE total_amount < 10000;

                                          QUERY PLAN                                           
-----------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings  (cost=1180.01..15413.42 rows=62913 width=21)
   Recheck Cond: (total_amount < '10000'::numeric)
   ->  Bitmap Index Scan on bookings_total_amount_idx  (cost=0.00..1164.28 rows=62913 width=0)
         Index Cond: (total_amount < '10000'::numeric)
(4 rows)
```

Выбран метод доступа `Bitmap Scan`. Он состоит из двух узлов:
- `Bitmap Index Scan` - читает индекс и строит битовую карту
- `Bitmap Heap Scan` - читает табличные страницы, используя построенную карту

Обратите внимание, что карта должна быть построена полностью, прежде чем ее можно будет использовать.


## Объединение битовых карт

Кроме того, что битовая карта позволяет избежать повторных чтений табличных страниц, с ее помощью можно объединить несколько условий.

```sql
EXPLAIN (costs off)
SELECT * FROM bookings WHERE total_amount < 10000 OR total_amount > 100000;

                                        QUERY PLAN                                         
-------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings
   Recheck Cond: ((total_amount < '10000'::numeric) OR (total_amount > '100000'::numeric))
   ->  BitmapOr
         ->  Bitmap Index Scan on bookings_total_amount_idx
               Index Cond: (total_amount < '10000'::numeric)
         ->  Bitmap Index Scan on bookings_total_amount_idx
               Index Cond: (total_amount > '100000'::numeric)
(7 rows)
```

Здесь сначала были построены две битовые карты - по одной на каждое условие, а затем объединены побитовой операцией "или".

Таким же образом могут быть использованы и разные индексы.
```sql
EXPLAIN (costs off)
SELECT * FROM bookings WHERE total_amount < 10000 OR book_date = bookings.now() - INTERVAL '1 day';

                                                                  QUERY PLAN                                                                   
-----------------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings
   Recheck Cond: ((total_amount < '10000'::numeric) OR (book_date = ('2017-08-15 15:00:00+00'::timestamp with time zone - '1 day'::interval)))
   ->  BitmapOr
         ->  Bitmap Index Scan on bookings_total_amount_idx
               Index Cond: (total_amount < '10000'::numeric)
         ->  Bitmap Index Scan on bookings_book_date_idx
               Index Cond: (book_date = ('2017-08-15 15:00:00+00'::timestamp with time zone - '1 day'::interval))
(7 rows)
```

