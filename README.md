# Kittygram

![Main Kittygram workflow](https://github.com/Andraboba/kittygram_final/actions/workflows/main.yml/badge.svg)

Kittygram — социальная сеть для любителей котиков. Пользователи регистрируются,
добавляют своих котиков с фотографиями, указывают цвет, год рождения и
достижения питомцев, а также смотрят котиков других пользователей.

Проект запускается в Docker-контейнерах. При пуше в ветку `main` GitHub Actions
проверяет код, запускает тесты, собирает образы и публикует их на Docker Hub,
а затем отправляет уведомление в Telegram.

## Стек технологий

- Python 3.10, Django 5.1, Django REST Framework, Djoser
- PostgreSQL 13
- Gunicorn
- Nginx 1.22
- React 17 (фронтенд)
- Docker, Docker Compose
- GitHub Actions, ruff

## Архитектура

| Контейнер  | Образ                         | Назначение                                  |
|------------|-------------------------------|---------------------------------------------|
| `db`       | `postgres:13`                 | База данных                                 |
| `backend`  | `username/kittygram_backend`  | Django + Gunicorn, порт 8000                |
| `frontend` | `username/kittygram_frontend` | Сборка React, копирует файлы в volume static |
| `gateway`  | `username/kittygram_gateway`  | Nginx, порт 9000 → 80                       |

Volumes:

- `pg_data` — данные PostgreSQL;
- `static` — статика фронтенда и бэкенда (доступна контейнеру `gateway`);
- `media` — файлы, загруженные пользователями (доступна `backend` и `gateway`).

Nginx проксирует `/api/` и `/admin/` в контейнер `backend`, раздаёт `/media/`
и статику, остальные запросы отдаёт фронтенду (`index.html`).

## Как запустить проект локально

1. Клонируйте репозиторий:

   ```bash
   git clone https://github.com/Andraboba/kittygram_final.git
   cd kittygram_final
   ```

2. Создайте файл `.env` по образцу `.env.example` и заполните его своими
   значениями (обязательно задайте `SECRET_KEY` и `POSTGRES_PASSWORD`):

   ```bash
   cp .env.example .env
   ```

3. Соберите и запустите контейнеры:

   ```bash
   docker compose -p kittygram up -d --build
   ```

4. Выполните миграции, соберите статику бэкенда и скопируйте её в volume:

   ```bash
   docker compose -p kittygram exec backend python manage.py migrate
   docker compose -p kittygram exec backend python manage.py collectstatic --noinput
   docker compose -p kittygram exec backend cp -r /app/collected_static/. /backend_static/static/
   ```

5. Создайте суперпользователя:

   ```bash
   docker compose -p kittygram exec backend python manage.py createsuperuser
   ```

Проект будет доступен по адресу <http://localhost:9000/>,
админка — <http://localhost:9000/admin/>, API — <http://localhost:9000/api/>.

Остановить проект: `docker compose -p kittygram down`
(с удалением данных: `docker compose -p kittygram down -v`).

### Запуск из готовых образов Docker Hub

В файле `docker-compose.production.yml` замените `username` на свой логин на
Docker Hub, затем выполните:

```bash
docker compose -f docker-compose.production.yml -p kittygram up -d
```

и повторите шаги 4–5, добавив `-f docker-compose.production.yml` к командам.

## Переменные окружения

| Переменная             | Описание                                         |
|------------------------|--------------------------------------------------|
| `POSTGRES_DB`          | Имя базы данных                                  |
| `POSTGRES_USER`        | Пользователь PostgreSQL                          |
| `POSTGRES_PASSWORD`    | Пароль PostgreSQL                                |
| `DB_HOST`              | Хост БД (в Docker — `db`)                        |
| `DB_PORT`              | Порт БД (`5432`)                                 |
| `SECRET_KEY`           | Секретный ключ Django                            |
| `DEBUG`                | Режим отладки (`True`/`False`)                   |
| `ALLOWED_HOSTS`        | Разрешённые хосты через запятую                  |
| `CSRF_TRUSTED_ORIGINS` | Доверенные источники для CSRF через запятую      |

Если `POSTGRES_DB` не задана, Django использует SQLite (удобно для разработки
без Docker).

## CI/CD (GitHub Actions)

Workflow описан в `.github/workflows/main.yml` (копия — `kittygram_workflow.yml`
в корне). При пуше в `main` выполняются jobs:

1. `tests` — проверка кода бэкенда `ruff check backend/` и запуск тестов Django
   против сервиса PostgreSQL 13;
2. `frontend_tests` — `npm ci` и `npm run test` для фронтенда;
3. `build_and_push_to_docker_hub`, `build_frontend_and_push_to_docker_hub`,
   `build_gateway_and_push_to_docker_hub` — сборка образов и публикация на
   Docker Hub (`<DOCKER_USERNAME>/kittygram_backend`, `kittygram_frontend`,
   `kittygram_gateway`);
4. `send_message` — уведомление в Telegram.

### Секреты репозитория

Добавьте в GitHub: **Settings → Secrets and variables → Actions →
New repository secret**:

| Секрет            | Значение                                                              |
|-------------------|-----------------------------------------------------------------------|
| `DOCKER_USERNAME` | Логин на Docker Hub                                                   |
| `DOCKER_PASSWORD` | Access Token Docker Hub (Account settings → Personal access tokens, права Read & Write) |
| `TELEGRAM_TO`     | ID вашего Telegram-аккаунта (можно узнать у бота @userinfobot)       |
| `TELEGRAM_TOKEN`  | Токен Telegram-бота (выдаёт @BotFather)                              |

После добавления секретов перезапустите workflow (Actions → нужный запуск →
Re-run all jobs) или сделайте новый пуш в `main`.

## Проверка автотестами Практикума

Создайте в корне файл `tests.yml` (шаблон уже лежит в репозитории) и заполните
его своими данными, затем выполните в корне проекта `pytest`.

## Автор

Andraboba — <https://github.com/Andraboba>
