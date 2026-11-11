# Контракты сервисов и фоновых задач

Читать вместе с [PLAN.md](../../PLAN.md) и [DATA_MODEL.md](DATA_MODEL.md).
Эта спецификация задаёт публичное поведение. JSON Schema/OpenAPI и примеры в contracts/ должны соответствовать ей.

## C1. Общие форматы HTTP

- JSON UTF-8. Идентификаторы — UUID, даты — ISO 8601 UTC.
- GET читает состояние. Проверка LM Studio, запуск анализа и экспорт используют POST.
- Длительная операция возвращает 202 с job_id; состояние доступно по GET /api/jobs/{id}.
- Успешная синхронная операция возвращает data и meta; список также возвращает next_cursor.
- PATCH изменяемой сущности требует expected_version; устаревшая версия — 409.
- POST создания задания, upload и complete поддерживают Idempotency-Key. Laravel хранит request hash и исходный ответ.
- Повтор с тем же ключом и телом возвращает исходный результат; с другим телом — 409 IDEMPOTENCY_CONFLICT.
- Списки: cursor, limit (по умолчанию 100, максимум 200), разрешённые sort/filter. Фильтрация на сервере.
- Ошибка: error.code, error.message_ru, error.retryable, error.action, error.field_errors, error.correlation_id.
- Для 500/503 UI не должен выдавать успешный toast. Технический trace доступен в логах по correlation_id.

HTTP-коды: 401 — отсутствует сервисная авторизация; 403 — нет доступа; 404 — сущность отсутствует;
409 — версия/аренда/состояние конфликтует; 413 — превышен лимит; 422 — неверные параметры;
429 — временный лимит; 503 — зависимость недоступна. Частичный успех массовой операции отражается в items, а не скрывается за общим «готово».

## C2. Публичное API Laravel

| Метод и путь | Обязанность |
| --- | --- |
| GET /api/bootstrap | Локальная сессия, настройки интерфейса, доступные capabilities и лимиты |
| GET/POST /api/collections | Список и создание коллекции |
| GET/PATCH/DELETE /api/collections/{id} | Чтение, правка с version, мягкое удаление |
| POST /api/collections/{id}/restore | Восстановление |
| GET /api/collections/{id}/images | Галерея с фильтрами и cursor |
| POST /api/collections/{id}/imports | Создать пакет импорта |
| POST /api/imports/{id}/files | Один файл и его устойчивый upload_key; 202 после записи во временный путь |
| GET /api/imports/{id} | Переданные, проверенные, ошибочные, пропущенные элементы |
| GET/PATCH/DELETE /api/images/{id} | Метаданные/решение пользователя/мягкое удаление |
| POST /api/images/{id}/restore | Восстановить |
| POST /api/collections/{id}/references | Установить эталонные изображения |
| POST /api/collections/{id}/analysis | Создать задачу выбранных этапов, без неявной генерации captions |
| POST /api/collections/{id}/recommendations | Рассчитать варианты набора; вернуть job_id |
| GET/POST /api/collections/{id}/drafts | Список и создание наборов |
| GET/PATCH /api/drafts/{id} | Настройки черновика с version |
| PATCH /api/drafts/{id}/items/{item_id} | Выбор, закрепление, transform, ссылка на caption |
| POST /api/drafts/{id}/transforms | Фоновая подготовка производных изображений |
| GET/PATCH /api/drafts/{id}/items/{item_id}/caption | Чтение и новая ручная ревизия |
| POST /api/drafts/{id}/captions | Генерация: missing / automatic_only / selected_replace |
| POST /api/caption-proposals/{id}/apply | Применить предложение с expected_version |
| POST /api/drafts/{id}/bulk-preview | Посчитать изменения по зафиксированному набору IDs/версий |
| POST /api/drafts/{id}/bulk-apply | Применить preview_id; изменившиеся элементы вернуть конфликтами |
| POST /api/drafts/{id}/preflight | Проверка снимка черновика; job_id |
| POST /api/drafts/{id}/publish | Опубликовать проверенный снимок; stale preflight → 409 |
| GET /api/revisions/{id} | Неизменяемая версия и её состав |
| POST /api/revisions/{id}/fork | Создать редактируемый черновик |
| POST /api/revisions/{id}/exports | Собрать папку или ZIP |
| GET/DELETE /api/exports/{id} | Состояние/мягкое удаление конкретного экспорта |
| POST /api/exports/{id}/restore | Восстановить экспорт |
| GET /api/jobs; GET /api/jobs/{id} | Центр задач и snapshot одной задачи |
| POST /api/jobs/{id}/pause, /resume, /cancel, /retry-failed | Команды управления, разрешённые для текущего состояния |
| GET/PATCH /api/settings/lm-studio | Секрет в ответ не возвращается; флаг token_configured |
| POST /api/settings/lm-studio/test | Проверка API из worker |
| POST /api/settings/lm-studio/vision-test | Проверка изображения через общий GPU-слот |
| GET /api/health | Состояние зависимостей, время последней проверки, причина |
| POST /api/realtime/ticket | Короткоживущий билет подписки с разрешёнными комнатами |
| GET /api/media/{artifact_or_asset_id} | Проверенная выдача файла/миниатюры |

Не принимать от браузера произвольный filesystem path. Экспорт получает root_id и безопасное relative_directory.
Пути /api/jobs/{id}/... в таблице команд имеют отдельные POST endpoints, а не строку со списком действий в одном URL.
Для captions режим missing обрабатывает только отсутствие текста; automatic_only — незаблокированные generated подписи; selected_replace — явно выбранные элементы после preview. Замена locked требует отдельного replace_locked=true и ожидаемых версий; разрешение не отменяет проверку позднего конфликта.
Точные request/response examples создать в contracts/openapi.yaml до реализации потребителей.

## C3. Сессия и закрытые интерфейсы

MVP рассчитан на одного локального владельца, без регистрации и многопользовательских ролей.
Laravel создаёт локальную same-origin сессию при bootstrap. Изменяющие запросы защищены CSRF; проверяются Host/Origin, cookie HttpOnly/SameSite.
Это режим локального приложения: включение доступа из LAN потребует отдельной полноценной аутентификации и не является переключателем MVP.

Билет Socket.IO выдаёт Laravel для текущей сессии: signed JWT, отдельный секрет, audience=realtime, срок 5 минут и разрешённые комнаты.
Node проверяет подпись, срок, audience и подписки; по истечении билет обновляется через API.
Не принимать имя комнаты от клиента без сверки с разрешениями. Все коллекции MVP принадлежат локальному владельцу, но произвольные служебные комнаты недоступны.

Внутренний worker API и Fastify /internal/events используют разные сервисные Bearer-токены.
Секреты не входят в события, browser bundle, query string и журналы.
Nginx публично проксирует только Socket.IO, а внутренние endpoints доступны через Compose. Origin-проверка не заменяет сервисный токен.

## C4. Протокол Python worker

| Endpoint на внутреннем listener Laravel | Результат |
| --- | --- |
| POST /internal/workers/register | worker_id, capabilities, server_time |
| POST /internal/workers/{id}/heartbeat | Признак живого процесса и текущие attempt IDs |
| POST /internal/tasks/claim | 200 с item/attempt/token/inputs; 204 если подходящей работы нет |
| POST /internal/attempts/{id}/heartbeat | Продление аренды + control=continue/pause_after_stage/cancel |
| POST /internal/attempts/{id}/progress | Измеримый прогресс и stage; контроль token/version |
| POST /internal/attempts/{id}/complete | Durable receipt для публикации результата; обычно 202 |
| POST /internal/attempts/{id}/fail | Код, retryable, безопасное сообщение и техническая ссылка |
| GET /internal/attempts/{id} | Состояние и ранее созданный receipt при потерянном ответе |
| GET /internal/integrations/lm-studio | Настройки текущей версии для авторизованной попытки |

Claim содержит неизменяемый снимок входов: item_id, attempt_id, lease_token, lease_expires_at,
input_fingerprint, stage, source/artifact keys, parameters, settings_version, resource_class, correlation_id.
Worker не подбирает другое изображение или профиль сам и не выполняет SQL.

Defaults для первой реализации, затем проверить на целевом ПК:

| Настройка | Начальное значение |
| --- | --- |
| Heartbeat попытки | каждые 10 секунд, отдельный поток/задача от inference |
| Lease running | 60 секунд, время считает PostgreSQL |
| Пустая очередь | ожидание 1–5 секунд с jitter |
| CPU-параллелизм | максимум 2 операции |
| GPU-слоты приложения | 1, общий для CV GPU, captions и vision-test |
| Publishing deadline | 5 минут, затем reconciliation и диагностируемый повтор |
| Временные ошибки этапа | максимум 3 попытки, backoff с jitter |
| Передача файлов браузером | максимум 3 одновременных запроса |

Scheduler возвращает истёкший running item в очередь только после завершения предыдущей аренды.
Token старой попытки больше не даёт права изменять item. Повтор уже принятого complete возвращает прежний receipt.
Worker прекращает новую работу при потере аренды; сервер всё равно проверяет её независимо.

JobItem — единица аренды: один этап одного изображения. Родительский Job даёт общий прогресс.
Независимые этапы не получают другую аренду на один и тот же выход. Unique constraint и транзакция защищают claim.
Heartbeat процесса и heartbeat попытки — разные сигналы; живой worker может иметь зависшую попытку.

## C5. Общие ресурсы, retries и отмена

- Планировщик использует ResourceLease в PostgreSQL. Все запросы vision, включая пробные из UI, проходят этот механизм.
- Очередь обслуживается по коллекциям round-robin с учётом priority. Пробный запрос может стать следующим, но не прерывает текущую генерацию.
- CPU-работа выполняется независимо от недоступности LM Studio.
- Автоматически повторять безопасные временные ошибки: соединение не установлено, 429 с Retry-After, документированная временная ошибка сервера.
- Не повторять бесконечно повреждённые файлы, неверные параметры, 401/403 и неподдерживаемые capabilities.
- Read timeout или падение worker после отправки inference оставляет неопределённость: LM Studio могла продолжить вычисление. ResourceLease становится uncertain; новый inference не выдаётся автоматически по истечению обычной аренды.
- Для uncertain требуется подтверждённое завершение/остановка внешнего запроса либо явное действие пользователя после проверки LM Studio. GET списка моделей не доказывает, что GPU свободен.
- Circuit breaker после 3 подряд однозначно временных сбоев откладывает зависимые задачи на 60 секунд; затем разрешает одну безопасную пробу. После исчерпания попыток — ожидание действия пользователя.
- OOM ставит GPU-очередь на паузу с подсказкой освободить VRAM; модель/контекст автоматически не менять.
- Cancel запрещает новые попытки и публикацию поздних ответов. Если API модели не подтверждает остановку inference, UI показывает «Отменено в студии; модель ещё может завершать запрос».
- Graceful shutdown перестаёт claim новые items, сохраняет состояние и завершает либо освобождает текущие безопасные операции. Не удалять attempts, пока они могут понадобиться reconciliation.

## C6. События и snapshot

Fastify: POST /internal/events, проверка Bearer, schema_version, размера и схемы; ответ 200 означает принятие.
Дубликат event_id допустим; Node может хранить короткий dedup-кеш, но корректность от него не зависит.
Laravel отправляет только события зафиксированных транзакций.

Конверт:

    {
      "schema_version": 1,
      "event_id": "uuid",
      "type": "job.updated",
      "aggregate_type": "job",
      "aggregate_id": "uuid",
      "aggregate_version": 17,
      "collection_id": "uuid",
      "occurred_at": "UTC ISO 8601",
      "correlation_id": "uuid",
      "data": { "status": "running", "completed_items": 12, "total_items": 80 }
    }

Типы: job.updated, job.completed, job.failed, asset.updated, import.updated, draft.updated,
caption.proposed, export.updated, integration.updated.
Каждый тип получает JSON Schema и примеры. Версии разных aggregates не сравнивать.
Комнаты: collection:{id}, job:{id}, owner:notifications; право на них выдаёт Laravel.

Job snapshot содержит job version, control, counts, current stages, elapsed_seconds, nullable percent/eta_seconds,
и компактный список item IDs/status/version. Большие тексты ошибок и метаданные изображений выдаются отдельно.
Сервер вычисляет terminal_items и completed_images отдельно; успех и ошибки показаны раздельно. Общий процент terminal_items/total_items допустим только для зафиксированного списка этапов; это не оценка времени.
Граф и total_items фиксируются при создании задачи. Новый отбор или retry-failed создаёт связанный новый Job с собственным total; история исходного прохода остаётся.

Алгоритм Vue:

1. Подключиться, получить подтверждение подписки, начать временно буферизовать события.
2. Загрузить snapshot через API.
3. Применить snapshot, затем более новые события; устаревшие/повторные игнорировать.
4. При пропуске aggregate_version перечитать snapshot соответствующего объекта.
5. При reconnect повторить алгоритм; HTTP-ответы также применять с учётом version.
6. Если Socket.IO недоступен, опрашивать только активные jobs раз в 5 секунд. При восстановлении остановить fallback.
7. Для фоновой вкладки снизить частоту; при возврате обновить snapshot. Не держать отдельный polling на каждую карточку.

Progress-события объединять до 2 в секунду на job; смены состояния, ошибки и завершение не задерживать намеренно.
Payload до 64 KiB. Не передавать картинки/base64, секреты и stack trace.
Outbox delivered хранить 7 дней; JobEvent по умолчанию 30 дней. Пользовательские решения/CaptionRevision/экспорты этой очисткой не затрагиваются.
Временные отказы Node повторять с backoff до 60 секунд; после 10 сбоев показывать проблему, не удаляя pending.
Невалидную схему/авторизацию переводить в needs_attention с возможностью повторной отправки после исправления.
В Outbox используется собственная короткая delivery lease, чтобы два цикла scheduler не отправляли одну пачку одновременно.

## C7. Интеграция LM Studio

Настройки: openai_base_url (по умолчанию http://host.docker.internal:1234/v1), optional token,
model_id, native_api_base_url (необязательно), request parameters, version.
Нормализовать trailing slash и не добавлять второй /v1. Native URL определять по известному адаптеру либо явной настройке; не ломать пользовательский proxy path.
Разрешены HTTP/HTTPS; локальные адреса нужны для назначения программы. Не открывать LM Studio в сети автоматически.

Проверка API:
1. Laravel создаёт job с версией настроек.
2. Python запрашивает /models относительно OpenAI base URL; timeout подключения 5 секунд, списка 15 секунд.
3. Возвращает доступность, задержку, модели и отдельные ошибки DNS/connect/auth/URL/timeout.
4. Native GET /api/v1/models при поддержке возвращает loaded_instances/capabilities.
5. Без native API статус загрузки — unknown, а не false. /v1/models не гарантирует, что модель уже загружена.

Vision-test использует выбранную картинку и общий GPU-слот, /chat/completions, явно выбранный model_id.
Начальный inference timeout 180 секунд, настраиваемый по пробе. Отличать загрузку модели от обработки картинки, когда API позволяет.
Сохранять фактическую модель, квантование (если известно), prompt revision и параметры.
Structured output включать только после capability-пробы; ответ валидировать и по схеме, и на пригодность caption.
Синтаксически верный JSON не подтверждает, что модель правильно увидела изображение.

## C8. Диагностика и приёмка контрактов

В логах общие correlation_id, job_id, item_id, attempt_id, event_id; токены и полный текст запросов с изображениями не логировать.
Health различает liveness процесса, readiness API/БД и доступность внешней интеграции.
Healthcheck Docker не является перезапуском зависшего процесса: тайм-ауты и завершение безнадёжно зависшей попытки реализуются в worker.
Для contract tests использовать минимальные fixtures из contracts/examples/; они проверяют взаимодействие, конфликты версий, повторы и восстановление.

## C9. Маски и ComfyUI

Применять [R3 дополнения](REQUIREMENTS_ADDENDUM.md): общий GPU, неизвестный исход запроса, отсутствие слепого повтора и адресная отмена.
Новые публичные маршруты под /api:
- GET/PUT /settings/comfyui; POST /settings/comfyui/check — URL, возможности и проверка закреплённого workflow; секреты не возвращать.
- GET /comfyui/workflows — только зарегистрированные версии.
- GET/POST /assets/{id}/protection-revisions — маска как проверенный artifact ID, размеры/отступ и expected_version.
- POST /draft-items/{id}/transform-preview — варианты planner с причиной и fingerprint входов.
- POST /draft-items/{id}/outpaint — 202 + job_id; workflow revision, preview fingerprint, expected_version, idempotency key.
- POST /draft-items/{id}/result-review — принять/отклонить конкретный artifact/QA revision; конфликт 409.
- POST /collections/{id}/archive и /unarchive — обратимая смена состояния с проверкой версии.

Применяются прежние авторизация, CSRF, ошибки и optimistic locking. Клиент не передаёт файловый путь или исполняемый workflow.
До реализации закрепить request/response JSON Schema для каждого маршрута в contracts/.
Worker-адаптер ComfyUI реализует check, submit, status, fetch_result, cancel_if_supported; конкретные внешние endpoints проверяются по установленной версии на S09a.
Сохранять внешний ID сразу после отправки; отсутствие ID после timeout — unknown, а не failed_retryable.
События protection.updated, generation.updated, result.review_required и result.reviewed используют C6 envelope и snapshot; маски/изображения передавать защищёнными ссылками, не в socket payload.

