# Статистика

## Число строк

Посмотрим на статистику на примере таблицы рейсов flights.
Значение параметра, управляющего размером статистики, по умолчанию равно 100:
```sql
SHOW default_statistics_targer;
```

Уменьшим его до 10 в целях демонстрации:
```sql
SET default_statistics_targer = 10;
```

```sql
ANALYZE;
```

Поскольку при анализе таблицы учитывается 300 * default_statistics_target строк, то оценки, как правило, не будут абсолютно точными.

Теперь посмотрим на оценку кардинальности в простом случае - запрос без предикатов.
```sql
EXPLAIN SELECT * FROM flights;
```

Точное значение:
```sql
SELECT count(*) FROM flights;
```

Оптимизатор получает значение из pg_class:
```sql
SELECT reltuples, relpages FROM pg_class WHERE relname = 'flights';
```


## Доля неопределенных значений

Часть рейсов еще не оправились, поэтому время вылета для них не определено:
```sql
EXPLAIN SELECT * FROM flights WHERE actual_departure IS NULL;
```

Точное значение:
```sql
SELECT count(*) FROM flights WHERE actual_departure IS NULL:
```

Оценка оптимизатора получена как общее число строк, умноженное на долю NULL-значений:
```sql
SELECT 214867 * null_frac FROM pg_stats WHERE tablename = 'flights' AND attname = 'actual_departure';
```


## Наиболее частые значения

Если значение есть в списке наиболее частых значений, то селективность можно узнать непосредственно из статистики.
Пример (Шереметьево):
```sql
EXPLAIN SELECT * FROM flights WHERE departure_airport = 'SVO';
```

Точное значение:
```sql
SELECT count(*) FROM flights WHERE departure_airport = 'SVO';
```


Вот как выглядит список наиболее частых значений и частота их встречаемости:
```sql
SELECT most_common_vals, most_common_freqs FROM pg_stats WHERE tablename = 'flights' AND attname = 'departure_airport' \gx
```

Максимальное число значений определяется параметром default_statistics_target, который в нашем случае равен 10.
Значение по умолчанию 100 немного не хватило бы, чтобы хранить все 104 значения.

Кардинальность вычисляется как число строк, умноженное на частоту значения:
```sql
SELECT 214867 * s.most_common_freqs[array_position((s.most_common_vals::text::text[]), 'SVO')]
FROM pg_stats s WHERE s.tablename = 'flights' AND s.attname = 'departure_airport';
```


## Число уникальных значений

Если же указанного значения нет в списке наиболее частых, то оно вычисляется исходя из предположения,
что все данные (кроме наиболее частых)  распределены равномерно.

Например, в списке частых значений нет Владивостока.
```sql
EXPLAIN SELECT * FROM flights WHERE departure_airport = 'VVO';
```

Точное значение:
```sql
SELECT count(*) FROM flights WHERE departure_airport = 'VVO';
```

Для получения оценки вычислим сумму частот наиболее частых значений:
```sql
SELECT sum(f) FROM pg_stats s, unnest(s.most_common_freqs) f WHERE s.tablename = 'flights' AND s.attname = 'departure_airport';
```

На менее частые значения приходятся оставшиеся строка.
Поскольку мы исходим из предположения о равномерности распределения менее частых значений,
селективность будет равна 1/nd, где nd - число уникальных значений:
```sql
SELECT n_distinct FROM pg_stats s WHERE s.tablename = 'flights' AND s.attname = 'departure_airport';
```

Учитывая, что из этих значений 10 входят в список наиболее частых, и нет неопределенных значений, получаем следующую оценку:
```sql
SELECT 214867 * (1 - 0.421667) / (103 - 10);
```















```sql

```