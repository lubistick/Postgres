# Индексное сканирование

Рассмотрим таблицу бронирований:
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
Столбец `book_ref` является первичным ключом и для него автоматически был создан индекс `bookings_pkey`.

Проверим план запроса с поиском одного значения:
```sql
EXPLAIN SELECT * FROM bookings WHERE book_ref = 'CDE08B';

                                  QUERY PLAN                                   
-------------------------------------------------------------------------------
 Index Scan using bookings_pkey on bookings  (cost=0.43..8.45 rows=1 width=21)
   Index Cond: (book_ref = 'CDE08B'::bpchar)
(2 rows)
```
План:
- `Index Scan` - доступ по индексу, указано имя использованного индекса
- `Index Cond` - условие доступа

## Как формируется стоимость?

Первое число `0.43` - оценка ресурсов для спуска к листовому узлу.
Она зависит от логарифма количества листовых узлов (количества операций сравнения, которые надо выполнить) и от высоты дерева.
При оценке считается, что необходимые страницы окажутся в кэше и оцениваются только ресурсы процессора: цифра получается небольшой.

Спуститься от корня индекса до листовой странички, после этого мы получим ссылку на табличную строку и будем готовы ее извлекать.

Второе число `8.45` - оценка чтения необходимых листовых страниц индекса и табличных страниц.
Не стоит забывать, что один узел `Index Scan` подразумевает обращение и к индексу, и к таблице.

В данном случае, поскольку индекс уникальный, будет прочитана одна индексная страница и одна табличная.
Каждое из чтений оценивается параметром:
```sql
SELECT current_setting('random_page_cost');

 current_setting 
-----------------
 4               
(1 row)
```

Его значение по умолчанию больше, чем `seq_page_cost`, поскольку произвольный доступ стоит дороже.
Хотя для SSD-дисков этот параметр следует существенно уменьшать (до `2`, до `1.5` или даже до `1.1`).

Итого получаем `8`, и еще немного добавляет оценка процессорного времени на обработку строк.

## Дополнительные условия

Дополнительные условия, которые можно проверить только по таблице, отображаются в отдельной строке `Filter`:
```sql
EXPLAIN SELECT * FROM bookings WHERE book_ref = 'CDE08B' AND total_amount > 1000;

                                  QUERY PLAN                                   
-------------------------------------------------------------------------------
 Index Scan using bookings_pkey on bookings  (cost=0.43..8.45 rows=1 width=21) 
   Index Cond: (book_ref = 'CDE08B'::bpchar)                                   
   Filter: (total_amount > '1000'::numeric)                                    
(3 rows)
```


## Параллельное индексное сканирование

Индексное сканирование может выполняться в параллельном режиме.
В качестве примера найдем сумму одной четверти всех бронирований:

```sql
EXPLAIN SELECT sum(total_amount) FROM bookings WHERE book_ref < '400000';

                                                    QUERY PLAN                                                    
------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=16861.86..16861.87 rows=1 width=32)
   ->  Gather  (cost=16861.64..16861.85 rows=2 width=32)
         Workers Planned: 2
         ->  Partial Aggregate  (cost=15861.64..15861.65 rows=1 width=32)
               ->  Parallel Index Scan using bookings_pkey on bookings  (cost=0.43..15312.03 rows=219843 width=6)
                     Index Cond: (book_ref < '400000'::bpchar)
(6 rows)
```
Аналогичный план мы уже видели при параллельном последовательном сканировании,
но в данном случае данные читаются с помощью индекса - узел `Parallel Index Scan`.

Стоимость обусловлена двумя составляющими.
Во-первых, стоимость доступа к `1/4` табличных страниц, поделенных между процессами - аналогично последовательному сканированию:
```sql
SELECT
  round(
    (relpages / 4.0) * current_setting('seq_page_cost')::real +
    (reltuples / 4.0) / 2.4 * current_setting('cpu_tuple_cost')::real
  )
FROM pg_class WHERE relname = 'bookings';

 round 
-------
  5561
(1 row)
```


