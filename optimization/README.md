# Демонстрационная база данных

Заходим [на сайт авторов](https://postgrespro.ru/education/demodb) и скачиваем самый большой дамп.
Или скачиваем [по этой ссылке](https://edu.postgrespro.ru/demo-big.zip).

## Локальный запуск

Подразумевается что у нас установлен `Docker`.
Распаковываем архив с дампом, например в `/home/lubis/postgres-demo/demo-big-20170815.sql`

Запускаем контейнер с `Postgres`, обязательно прикручиваем volume:

```bash
docker run -e POSTGRES_PASSWORD=postgres -v /home/lubis/postgres-demo:/dump -d --name postgres-demo postgres:14-alpine
```

Заходим внутрь контейнера:

```bash
docker exec -ti postgres-demo sh
```

Накатываем дамп:

```bash
psql -U postgres < /home/demo-big-20170815.sql
```

Ждем... Готово.
