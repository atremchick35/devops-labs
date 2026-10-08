# devops-labs

Flask + Nginx + PostgreSQL + Redis в Docker Compose (домашнее задание второй пары по Docker).
Подробный отчёт с командами и выводом — [REPORT.md](REPORT.md).

```bash
cp .env.example .env                                   # задайте свой POSTGRES_PASSWORD
docker compose -f compose.yaml up -d --build --wait    # базовый запуск
curl localhost:18090/                                  # счётчик (Redis)
curl --get --data-urlencode 'text=hello' localhost:18090/notes/add
curl localhost:18090/notes                             # заметки (PostgreSQL)
docker compose watch                                   # режим разработки (compose.override.yaml)
docker compose down                                    # том с заметками сохраняется
```
