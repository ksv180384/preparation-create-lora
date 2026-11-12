# LoRA Dataset Studio

Локальная студия подготовки изображений: Vue 3 + Laravel + Python + PostgreSQL +
Fastify/Socket.IO. Целевой адрес http://localhost:7676.
Состояние: выполнена подготовка S00; приложение и Compose создаются начиная с LDS-004.

## Документы

- [План](PLAN.md), [этапы](docs/plan/STAGES.md), [прогресс](docs/plan/PROGRESS.md)
- [Среда](docs/environment.md), [версии](docs/versions.md)
- [Архитектура](docs/architecture/README.md), [решения](docs/decisions/README.md)
- [Правила разработки](CONTRIBUTING.md), [Trello](docs/plan/TRELLO_BACKLOG.md)

## Работа в WSL

Открыть Ubuntu, затем:

```sh
cd /home/ksv180384/projects/preparation-create-lora.local
git status --short
docker --version
docker compose version
docker info
cp -n .env.example .env
```

.env.example не содержит рабочих секретов. На S01 сгенерировать APP_KEY и отдельные
пароли/служебные токены; не использовать пустые значения для запуска.

После появления compose.yaml на S01 команды из корня:
```sh
docker compose up -d --build
docker compose logs -f --tail=100
docker compose down
docker compose exec api php artisan migrate
```
Сейчас эти команды запуска приложения ещё неприменимы: compose.yaml отсутствует.
Не использовать down -v для обычной остановки. Данные хранить в WSL, originals отдельно от attempts.
LM Studio и ComfyUI работают на Windows, URL задаются позднее в UI. localhost внутри контейнера
не является Windows-хостом; использовать настроенный адрес (обычно host.docker.internal)
и проверять его из Python-контейнера.

## Структура

apps/api — Laravel; apps/web — Vue; services/worker — Python; services/realtime — Fastify.
infra/docker и infra/nginx — инфраструктура; contracts — API/events; profiles — профили.
scripts — команды обслуживания; docs — спецификации и решения. src/ сохранён как исходный пустой каталог.
