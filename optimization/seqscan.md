# Последовательное сканирование

При последовательном чтении всех страниц:
- страницы читаются в кэш (используется буферное кольцо)
- проверяется видимость версий строк
- данные возвращаются в произвольном порядке
- время сканирования зависит от физического размера файла

В плане выполнения запроса последовательное сканирование представлено узлом `Seq Scan`:
```sql
EXPLAIN SELECT * FROM flights;

                           QUERY PLAN                           
----------------------------------------------------------------
 Seq Scan on flights  (cost=0.00..4772.67 rows=214867 width=63)
(1 row)
```

Значения в скобках:
- `cost` - оценка стоимости
- `rows` - оценка числа строк, которые планировщик предполагает выбрать
- `width` - оценка размера одной записи в байтах, не стоит обращать на нее внимание, она нужна планировщику, чтобы знать, сколько памяти выделять

Стоимость `cost` указывается в условных единицах и состоит из двух компонент:
- Начальная стоимость вычисления узла (ресурсы, которые надо потратить на подготовительные действия). Для последовательного сканирования это ноль - чтобы возвращать данные, подготовки не требуется.
- Полная стоимость для получения всех данных (ресурсы, которые надо потратить, чтобы полностью отработал узел).

## Как рассчитывается стоимость?

У планировщика есть математическая модель, по которой он считает эти числа. Она учитывает:
- дисковый ввод-вывод
- ресурсы процессора

### Оценка дискового ввода-вывода

Рассчитывается как произведение числа СТРАНИЦ (не строк) в таблице на условную стоимость чтения одной страницы:
```sql
SELECT
  relpages,
  current_setting('seq_page_cost'),
  relpages * current_setting('seq_page_cost')::real AS total
FROM pg_class WHERE relname = 'flights';

 relpages | current_setting | total 
----------+-----------------+-------
     2624 | 1               |  2624
(1 row)
```

Стоимость чтения при последовательном сканировании равна единице. Обычно эту оценку никогда не меняют, чтобы была некая точка отсчета.

### Оценка ресурсов процессора

Складывается из стоимости обработки каждой строки:
```sql
SELECT
  reltuples,
  current_setting('cpu_tuple_cost'),
  reltuples * current_setting('cpu_tuple_cost')::real AS total
FROM pg_class WHERE relname = 'flights';

 reltuples | current_setting |  total  
-----------+-----------------+---------
    214867 | 0.01            | 2148.67
(1 row)
```

Сумма чисел `2624` и `2148.67` и есть общая стоимость `4772.67`.

## Последовательное сканирование и агрегация

Более сложный запрос с агрегатной функцией:
```sql
EXPLAIN SELECT COUNT(*) FROM seats;

                          QUERY PLAN                           
---------------------------------------------------------------
 Aggregate  (cost=24.74..24.75 rows=1 width=8)
   ->  Seq Scan on seats  (cost=0.00..21.39 rows=1339 width=0)
(2 rows)
```

План состоит из двух узлов:
- снизу `Seq Scan` - последовательный доступ к таблице, поскольку нужны все строки
- сверху `Aggregate` - подсчет количество строк

Узел `Aggregate` получает данные от нижнего узла `Seq Scan`.
Начальная стоимость агрегации практически равна полной.
Это означает, что узел не может выдать результат, пока не обработает все данные.

### Оценка узла агрегации

Разница между нижней оценкой `24.74` для `Aggregate` и верхней оценкой `21.39` для `Seq Scan` - стоимость работы узла `Aggregate`.
Вычисляется исходя из оценки ресурсов процессора на выполнение элементарной операции:
```sql
SELECT
  reltuples,
  current_setting('cpu_operator_cost'),
  reltuples * current_setting('cpu_operator_cost')::real AS total
FROM pg_class WHERE relname = 'seats';

 reltuples | current_setting |   total   
-----------+-----------------+-----------
      1339 | 0.0025          | 3.3474998
(1 row)
```

Число примерно `3.35` - и есть разница между `24.74` и `21.39`.


## Параллельные планы

Есть ведущий процесс, который занимается выполнением запроса.
Ведущий процесс может запускать рабочие процессы себе в помощь.
И некую часть плана эти процессы могут выполнять одновременно.
Потом они передают результат работы ведущему процессу.

Посмотрим на план с параллельным последовательным сканированием:
```sql
EXPLAIN SELECT COUNT(*) FROM bookings;

                                         QUERY PLAN                                         
--------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=25442.58..25442.59 rows=1 width=8)
   ->  Gather  (cost=25442.36..25442.57 rows=2 width=8)
         Workers Planned: 2
         ->  Partial Aggregate  (cost=24442.36..24442.37 rows=1 width=8)
               ->  Parallel Seq Scan on bookings  (cost=0.00..22243.29 rows=879629 width=0)
(5 rows)
```

План:
- `Parallel Seq Scan` - параллельное последовательное сканирование
- `Partial Aggregate` - частичная агрегация
- `Workers Planned` - сколько рабочих процессов было запланировано к запуску (их `2` плюс еще один ведущий, итого `3`)
- `Gather` - разделяет последовательную часть плана и параллельную
- `Finalize Aggregate` - ведущий процесс в одиночку заканчивает агрегацию

Подробнее поговорим про каждый узел плана.

### Parallel Seq Scan

Страницы читаются последовательно, но разными процессами.

С точки зрения дискового ввода-вывода ничего не меняется - таблицу все равно придется прочитать страница за страницей.

А затраты процессора становятся меньше.
Обработка прочитанных страниц выполняется параллельно.
Всего запланировано `2` рабочих процесса (каждый процесс выполняется в своем ядре процессора).
Также часть работы выполнит ведущий процесс, поэтому общее число строк делится на `2.4`
(доля ведущего процесса уменьшается с ростом числа рабочих процессов):
```sql
SELECT round(
  relpages * current_setting('seq_page_cost')::real +
  reltuples * current_setting('cpu_tuple_cost')::real / 2.4
)
FROM pg_class WHERE relname = 'bookings';

 round 
-------
 22243
(1 row)
```

В поле `rows` показана оценка числа строк `879629`, которые обработает один рабочий процесс:
:
```sql
SELECT round(reltuples / 2.4) AS "rows" FROM pg_class WHERE relname = 'bookings';

  rows  
--------
 879629
(1 row)
```

### Partial Aggregate

Узел выполняет агрегацию данных, полученных рабочим процессом, т.е. в данном случае подсчитывает количество строк.
Оценка выполняется уже известным образом и добавляется к оценке сканирования таблицы:
```sql
SELECT round(
  reltuples / 2.4 * current_setting('cpu_operator_cost')::real
)
FROM pg_class WHERE relname = 'bookings';

 round 
-------
  2199
(1 row)
```
Сложим `2199` (затраты процессора на обработку `Partial Aggregate`) и `22243` (стоимость `Parallel Seq Scan`) - получим `24442` (стоимость `Partial Aggregate`).

Узел выполняется тоже в параллельном режиме.

### Gather

Узел выполняется на ведущем процессе. Он отвечает за запуск рабочих процессов и получение от них данных - 
каждый из параллельных процессов получил одно число, и им надо это число передать ведущему процессу.

Запуск процессов и пересылка каждой строки данных оцениваются как:
```sql
SELECT
  current_setting('parallel_setup_cost') parallel_setup_cost,
  current_setting('parallel_tuple_cost') parallel_tuple_cost
;

 parallel_setup_cost | parallel_tuple_cost 
---------------------+---------------------
 1000                | 0.1
(1 row)
```

В данном случае пересылается всего одна строка и основная стоимость приходится на запуск.

Сложим `24442` (стоимость `Partial Aggregate`), `1000` (затраты на запуск процессов) и `0.1` (стоимость пересылки одной строки) - получим `25442` (стоимость узла `Gather`).

### Finalize Aggregate

Ведущий процесс агрегирует полученные частичные агрегаты. Поскольку для этого надо сложить всего `3` числа, оценка минимальна.

### Workers Planned

Почему было запланировано `2` рабочих процесса?

Реальное количество процессов ограничивается количеством доступных ядер процессора. А также критериями ниже.

Общее количество фоновых рабочих процессов, которые могут быть в системе, определяется параметром:
```sql
SELECT current_setting('max_worker_processes');

 current_setting 
-----------------
 8
(1 row)
```

Из них некоторая часть отведена исключительно на параллельное выполнение запросов:
```sql
SELECT current_setting('max_parallel_workers');

 current_setting 
-----------------
 8
(1 row)
```

Максимальное число процессов, которые используются при одном выполнении запроса ограничены параметром:
```sql
SELECT current_setting('max_parallel_workers_per_gather');

 current_setting 
-----------------
 2
(1 row)
```

Количество параллельных процессов будет фиксировано, если указан параметр хранения `parallel_workers` для конкретной таблицы. Но не может превышать `max_parallel_workers_per_gather`.
Вот так можно изменить параметр:
```sql
ALTER TABLE bookings SET (parallel_workers = 4);

ALTER TABLE
```

Вернем обратно:
```sql
ALTER TABLE bookings RESET (parallel_workers);

ALTER TABLE
```

А если не указан `parallel_workers` - будет зависеть от размера таблицы. Проверим размер таблицы `bookings`:
```sql
SELECT pg_size_pretty(pg_table_size('bookings')) size, (SELECT COUNT(*) FROM bookings);

  size  |  count  
--------+---------
 105 MB | 2111110
(1 row)
```


Параллельный план НЕ состоится, если размер таблицы меньше параметра `min_parallel_table_scan_size`:
```sql
SELECT current_setting('min_parallel_table_scan_size');

 current_setting 
-----------------
 8MB
(1 row)
```

Далее с ростом размера таблицы будет увеличено количество рабочих процессов:
- если размер таблицы `8 MB` и более, то под нее запускается `1` рабочий процесс
- если размер таблицы увеличится в 3 раза (`24 MB`), процессов станет `2`
- если размер таблицы увеличится еще в 3 раза (`72 MB`), процессов станет `3`
- и т.д.


### Участие ведущего процесса в параллельном плане

По умолчанию ведущий процесс участвует в выполнении параллельной части плана:
```sql
SHOW parallel_leader_participation;

 parallel_leader_participation 
-------------------------------
 on
(1 row)
```

Если ведущий процесс становится узким местом, его можно разгрузить.
Тогда он будет занят только сбором данных с рабочих процессов и выполнением последней части плана:
```sql
SET parallel_leader_participation = off;

SET
```

Посмотрим на план:
```sql
EXPLAIN SELECT COUNT(*) FROM bookings;

                                         QUERY PLAN                                          
---------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=27641.65..27641.66 rows=1 width=8)
   ->  Gather  (cost=27641.44..27641.65 rows=2 width=8)
         Workers Planned: 2
         ->  Partial Aggregate  (cost=26641.44..26641.45 rows=1 width=8)
               ->  Parallel Seq Scan on bookings  (cost=0.00..24002.55 rows=1055555 width=0)
(5 rows)
```

Теперь общее число строк делится на 2:
```sql
SELECT round(reltuples / 2) AS "rows" FROM pg_class WHERE relname = 'bookings';

  rows   
---------
 1055555
(1 row)
```

Вернем значение по умолчанию для параметра `parallel_leader_participation`:
```sql
RESET parallel_leader_participation;

RESET
```


## Итоги

Если мы просто захотим прочитать все строки из таблицы, параллельное выполнение будет невыгодно.
Потому что рабочим процессам придется передавать все прочитанные строки ведущему процессу.
А передача данных между процессами - далеко не бесплатная вещь.

Но если мы выполняем агрегацию - параллельный план становится эффективным.
Потому что каждый из процессов выполняет у себя частичную агрегацию.
После этого им остается передать ведущему процессу одно число - результат агрегации.
А затем ведущий процесс, когда соберет со всех процессов результат - довыполнит агрегацию.
