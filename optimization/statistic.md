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

...












```sql

```