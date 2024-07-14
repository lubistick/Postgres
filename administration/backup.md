# Резервное копирование

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

Вот как выглядит таблица в выводе команды `COPY`:
```sql
COPY t TO STDOUT;

1       Hello
2
3       \N
```

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

