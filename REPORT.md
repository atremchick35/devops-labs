# Отчёт

Docker 27.3.1
Compose v2.29.7
dive 0.13.1
Linux-ВМ Docker Desktop
Образы: `python:3.14-slim`, `nginx:1.30-alpine`, `postgres:18`, `redis:8.8-alpine`

## 1. Запуск

У `db` и `cache` есть healthcheck, `app` использует их через `depends_on: condition: service_healthy`, порт публикует только `web`. Зависимости закреплены в `requirements.txt`, а в Dockerfile `pip install` выполняется до `COPY app/ .`.

```
$ docker compose up -d --build --wait
$ docker compose ps
$ curl localhost:18090/
$ curl localhost:18090/notes
```

## 2. Том и сохранность данных

`postgres:18` хранит данные в `/var/lib/postgresql/18/docker`, поэтому том `pgdata` подключён к `/var/lib/postgresql`. После `docker compose rm -f db` и `up -d db` контейнер сменился (ID `25510504…` → `0b1484ce…`), а заметка осталась.

```
$ docker compose exec db sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT * FROM notes;"'
  1 | control note: survives volume
```

Сохраняем заметки в PostgreSQL. Счётчик Redis временный: тома нет, запуск с `--save ""`, и после пересоздания `cache` он начинается с `visits: 1`. Его потерю допускаем.

## 3. Холодный бэкап и восстановление

```
$ docker compose down                         # остановлены app, web, cache, db; том сохранён
$ docker run --rm -v devops-hw_pgdata:/data:ro -v "$PWD/backups":/backup --entrypoint tar postgres:18 --numeric-owner -czpf /backup/pgdata-cold.tar.gz -C /data .
$ docker volume rm devops-hw_pgdata
$ docker volume create --label com.docker.compose.project=devops-hw --label com.docker.compose.volume=pgdata devops-hw_pgdata
$ docker run --rm -v devops-hw_pgdata:/data -v "$PWD/backups":/backup:ro --entrypoint tar postgres:18 --numeric-owner -xzpf /backup/pgdata-cold.tar.gz -C /data
$ docker compose up -d --wait
$ curl localhost:18090/notes                  -> 1: control note: survives volume
db-1 | database system was shut down at ...
```

Архив занимает 6.7 MB (1308 файлов) и лежит в `backups/` вне тома. Восстановление проверено на том же образе `postgres:18`.

## 4. Сети

`web` подключён только к `front`, `db` и `cache` — только к `back` (`internal: true`), а `app` — к обеим сетям.

```
$ docker compose exec web getent hosts db              -> (пусто, имя не найдено)
$ docker compose exec web nc -v -w 3 172.20.0.3 5432   -> Operation timed out
$ docker compose exec web nc -z app 8000               -> OPEN   (контроль: nc работает)
$ docker compose exec app python -c "...connect(db); redis.ping()"
app -> db: PostgreSQL 18.6   app -> cache PING: True
```

## 5. dive

| | before | after |
|---|---|---|
| размер | 1.21 GB | 162 MB |
| efficiency | 95.7 % | 97.6 % |
| wasted | 97 MB | 5.7 MB |
| CI | FAIL (exit 1) | PASS |

## dive в CI

`.github/workflows/dive.yml` запускает `CI=true dive`

## Очистка

`docker compose down -v` и `docker image rm devops-hw-app:*`
