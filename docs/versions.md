# Выбор версий — LDS-002

Решение от 12 ноября 2026. Это целевая матрица, не отчёт об установке.
Ранее записанные Docker 24, Node 20 и «CUDA 12.1 подтверждена» не соответствовали измерениям.
Предыдущий рабочий текст сохранён в archive/versions-before-s00.md.

| Компонент | Принятая линия / исходная версия | Фиксация |
| --- | --- | --- |
| PHP | 8.4, исходная 8.4.27 | S01: точный образ и digest после разрешения в registry |
| Laravel | 13.x | composer.lock при создании приложения S01 |
| Composer | 2.x | точный runtime/образ в S01 |
| Node | 24.22.3 LTS | .node-version; один runtime Vue tooling и Fastify |
| Vue / TypeScript | 3.x / совместимая стабильная TS | package-lock.json при scaffold |
| Vite / Router / Pinia | совместимые с Vue 3 и Node 24 | проверить peerDependencies, зафиксировать package-lock |
| Fastify / Socket.IO | 5.x / 4.x | server/client Socket.IO одной линии; npm lock |
| Python | 3.12.16 | .python-version; CPU worker сначала |
| PostgreSQL | 17.x | S01: актуальный patch и digest образа |
| Nginx | стабильная поддерживаемая линия | S01: exact tag/digest |
| PyTorch / CUDA runtime / CV | не выбраны | S08: матрица конкретной модели и реальная GPU-проба |
| LM Studio / ComfyUI | внешний установленный runtime | версии/workflow фиксировать при интеграции |

Не выдумывать lock-файлы и digest до разрешения зависимостей. На S00 зафиксированы линии
совместимости; установка, точные транзитивные версии и проверка контейнеров входят в S01.
Нельзя выдавать совместимость по документации за успешную сборку всех библиотек.
Python 3.12 выбран как консервативная база CV; конкретный inference пакет проверяется отдельно.

## Основания (проверено 12 ноября 2026)

- Laravel 13 требует PHP >=8.3: https://laravel.com/framework/docs/13.x/deployment
- Поддержка PHP: https://www.php.net/supported-versions.php ; исходная 8.4.27: https://www.php.net/downloads.php?version=8.4
- Node 24.22.3 LTS: https://nodejs.org/en/blog/release/v24.22.3
- Vue tooling: https://vuejs.org/guide/quick-start.html
- Vite требует Node 20.19+ / 22.12+: https://vite.dev/guide/
- Fastify LTS следует поддерживаемым Node LTS: https://github.com/fastify/fastify/blob/main/docs/Reference/LTS.md
- Socket.IO 4: https://socket.io/docs/v4/
- Python 3.12.16: https://www.python.org/downloads/release/python-31216/
- PostgreSQL 17 поддерживается до ноября 2029: https://www.postgresql.org/support/versioning/

## Правила обновлений

Коммитить composer.lock, package-lock.json и Python lock с точными зависимостями/хешами.
Не использовать latest и плавающие production-образы. Обновление — отдельная задача с
changelog, проверкой совместимости и обновлением матрицы. Не переносить автоматически
обновление модели/workflow на существующие датасеты.
