# Модель данных и сохранность результатов

Читать вместе с [основным планом](../../PLAN.md). Эта спецификация определяет значения сущностей и правила записи.
Имена таблиц использовать в snake_case, модели Laravel — PascalCase. Все связи и ограничения создаёт Laravel.

## D1. Общие правила

- Идентификаторы предметных сущностей — UUID; даты — UTC, отображение в часовом поясе пользователя.
- Изменяемые записи имеют целочисленную version, начинающуюся с 1. Каждая успешная предметная правка увеличивает её.
- SHA-256 используется для целостности и точных дублей. Perceptual hash и embedding — только сигналы сходства.
- Деньги и удалённое облачное хранилище не нужны. Ключ файла — относительный путь под известным корнем; абсолютный пользовательский путь в БД не становится разрешением доступа.
- JSON допустим для результатов анализа и конфигураций. Состояния, связи, версии, сроки аренды и ключи идемпотентности — отдельные индексируемые поля.
- Полные JSON Schema и миграции создаются на соответствующих этапах по этим правилам; необязательное поле не заменяет состояние ошибки.

## D2. Сущности

| Сущность | Минимальные данные и назначение |
| --- | --- |
| Collection | name, image_kind, description, version, deleted_at; самостоятельная коллекция персонажа |
| SourceFile | sha256, storage_key, byte_size, mime, width, height, original_metadata; неизменяемый оригинал |
| ImageAsset | collection_id, source_file_id, original_name, relative_import_path, import_batch_id, base_caption_revision_id, base_caption_locked, review_decision, version, deleted_at |
| ImportBatch / ImportItem | очередь загрузок, upload_key, временный путь, состояние передачи/проверки, ошибки, найденная TXT-пара |
| ReferenceImage | collection_id, image_asset_id, role; несколько выбранных эталонов, без изменения исходника |
| AnalysisResult | asset_id, stage, input_fingerprint, tool_id/revision, result_json, warnings, completed_at |
| ReviewDecision | asset_id, decision, corrected_person_count, reason, previous_decision, created_at; история ручного решения |
| RecommendationResult | collection_id, input_versions, strategy_revision, candidates/groups, proposals, coverage, reasons |
| DatasetDraft | collection_id, name, goal, trigger, profile_id/revision, settings_snapshot, version, base_revision_id |
| DraftItem | draft_id, asset_id, included, pinned, transform_spec, artifact_id, active_caption_id, caption_locked, caption_review_state, version |
| CaptionRevision | asset_id, draft_item_id (nullable), text, source, input_fingerprint, based_on_version, model/prompt metadata |
| CaptionProposal | target, caption_revision_id, conflict_reason; фоновое предложение, не ставшее активной подписью |
| DatasetRevision | draft_id, revision_number, frozen_settings, manifest_hash, published_at; неизменяемый снимок |
| DatasetRevisionItem | revision_id, asset_id, artifact_id, caption_revision_id, export_basename, hashes; неизменяемый состав |
| Export | revision_id, root_id, relative_path, format, status, manifest_hash, job_id, published_at |
| ExportPreflight | draft_id/version или revision_id, frozen_input, errors, warnings, checked_versions, created_at |
| ProcessingJob | kind, scope, status, control, counts, version, correlation_id, frozen_input, priority |
| ProcessingJobItem | job_id, target_id, stage, worker_kind, status, attempt_count, input_fingerprint, resource_class, retry_at |
| JobAttempt | item_id, token_hash, lease_expires_at, last_heartbeat_at, status, attempt_path, receipt_id, failure |
| Artifact | storage_key, sha256, size, dimensions, producer_attempt_id, input_fingerprint; принятый неизменяемый результат |
| ResourceLease | resource_name, attempt_id, expires_at, state; общий слот GPU |
| JobEvent | предметная история этапов и решений с correlation_id |
| OutboxEvent | event_id, payload, status, attempt_count, next_attempt_at, delivery_lease_until, published_at |
| IdempotencyRecord | scope, key, request_hash, response_status/body, expires_at |
| IntegrationSettings | URL LM Studio, зашифрованный токен, выбранная модель, version, timeouts |
| StorageRoot | label, container_path, display_windows_path/display_wsl_path, permissions |

CaptionRevision без draft_item_id — исходная импортированная/ручная подпись. Черновик получает свою активную ссылку и может её изменить независимо.
Новая правка создаёт CaptionRevision; предыдущие тексты не обновлять SQL UPDATE. Источник source: imported, manual, generated.
Блокировка принадлежит цели редактирования (ImageAsset или DraftItem), чтобы одна подпись могла независимо использоваться в разных наборах. Изменение блокировки увеличивает version цели.
Стратегия подписи и входной кадр принадлежат набору. Подпись одного профиля не должна автоматически менять другой профиль.

## D3. Обязательные ограничения

- SourceFile.sha256 уникален; отдельные ImageAsset могут ссылаться на него в разных коллекциях.
- Уникальны (collection_id, source_file_id) среди активных элементов, если пользователь не создаёт осознанную альтернативную запись; в первой версии повторный импорт возвращает существующий элемент.
- Уникальны (draft_id, asset_id), (draft_id, revision_number), (scope, idempotency_key) и OutboxEvent.event_id.
- AnalysisResult повторно используется по stage + input_fingerprint + tool revision; проверять существование и хеш зависимого Artifact.
- У каждого JobItem не более одной активной попытки. Получение аренды выполняется атомарно в БД.
- Правка состава/transform/активной подписи/подтверждения DraftItem увеличивает также DatasetDraft.version в той же транзакции.
- Preflight хранит frozen_input и проверенные версии ImageAsset/DraftItem/caption/profile. Publish повторно сверяет все зависимости; изменение решения по исходнику тоже инвалидирует preflight.
- Важные индексы: collection_id + deleted_at; draft_id; status + retry_at; lease_expires_at; outbox status + next_attempt_at.
- Внешние ключи опубликованных ревизий запрещают каскадное уничтожение файлов и CaptionRevision.
- Размеры, счётчики и версии неотрицательны; completed_items + failed_items + skipped_items + cancelled_items не превышает total_items. Счётчики изображений хранятся отдельно от счётчиков этапов.

## D4. Статусы не смешиваются

| Объект | Значения |
| --- | --- |
| Ручное решение ImageAsset | unreviewed, accepted, excluded, needs_review |
| Анализ отдельного этапа | отсутствует, queued, running, succeeded, failed, stale |
| Включение в набор | included=true/false, независимо от анализа |
| JobItem | queued, waiting_resource, running, retry_wait, publishing, succeeded, failed, skipped, cancelled |
| Job control | run, pause_requested, paused, cancel_requested |
| Job status | queued, running, waiting_resource, paused, cancelling, cancelled, completed, completed_with_errors, failed |
| JobAttempt | running, publishing, succeeded, failed, expired, cancelled |
| Export | queued, building, validating, ready, failed, cancelled, trashed |
| OutboxEvent | pending, sending, delivered, needs_attention |

Job status агрегируется сервером из item-статусов и control. Браузер не устанавливает статус напрямую.
completed: все элементы терминальны, нет ошибок и отмены; completed_with_errors: есть успешный результат и ошибки; failed: задача не дала полезного результата из-за ошибок.
cancelled: отмена завершена, частичные результаты остаются доступны. paused — пауза с сохранёнными результатами, а не ошибка.
После cancel_requested новые элементы не выдаются; после pause_requested выполняющиеся попытки доходят до безопасной границы.
Publishing тоже проверяет control: поздний результат отменённой попытки не публикуется.

## D5. Ручное редактирование и конфликты

1. GET возвращает значение и version.
2. PATCH обязан передать expected_version.
3. Laravel сравнивает version в транзакции. При совпадении сохраняет ревизию, увеличивает version, записывает JobEvent/OutboxEvent.
4. При несовпадении отвечает 409 VERSION_CONFLICT с актуальной версией. UI сохраняет несохранённый текст и предлагает сравнить.
5. При активации imported/manual подписи устанавливается caption_locked=true у DraftItem или base_caption_locked=true у ImageAsset.
6. Запрос генерации фиксирует expected caption version, transform fingerprint и prompt revision.
7. Ответ модели становится активным только при неизменившихся входах, действующей задаче и разрешённой замене. Для locked подписи требуется отдельное явное подтверждение replace_locked после preview; оно привязано к ожидаемой версии. Без него сохраняется CaptionProposal. Любая последующая ручная правка всё равно запрещает автоматическое применение ответа.
8. Кнопка «Применить предложение» — отдельная ручная операция с проверкой version.
9. Массовые правки сначала возвращают preview/diff и affected_count; применение использует тот же снимок версий.
10. Undo создаёт новую ревизию из старого значения. Историю не стирать.

При изменении crop автоматическая подпись становится stale; locked подпись сохраняется, но требует подтверждения соответствия.
Публикация не допускается, пока текущая правка в браузере не сохранена или явно отменена.

## D6. Публикация версии и редактирование экспорта

- Черновик можно свободно менять с контролем version.
- Publish фиксирует draft version, состав, версии подписей, transform и profile revision в одном согласованном снимке.
- Проверки не должны читать «самую новую подпись» после фиксации снимка.
- DatasetRevision и его элементы после публикации не меняются.
- Export копирует файлы конкретной ревизии. Не создавать изменяемые hardlink на исходники/артефакты.
- «Редактировать экспорт» создаёт DatasetDraft с base_revision_id. Пользователь видит, какие подписи/кадры изменились.
- Удаление экспорта убирает конкретную физическую копию в корзину, но не удаляет коллекцию и другие экспорты.
- Экспорт image+TXT без изменений можно заново создать из ревизии и получить те же хеши содержимого; хеш ZIP не обязан совпадать из-за метаданных архива.

## D7. Права файловой системы и защита от устаревшего worker

| Каталог | Laravel api/app-worker | Python |
| --- | --- | --- |
| originals | запись только при первичном приёме; далее чтение | read-only |
| attempts | чтение и удаление завершённого мусора | запись только в путь своей попытки |
| artifacts | публикация новых неизменяемых файлов | read-only |
| exports, staging публикации | запись | без записи |
| PostgreSQL volume | только PostgreSQL | нет доступа |

Использовать согласованные UID/GID и отдельные mounts. Read-only originals и artifacts — обязательное свойство Compose, а не только соглашение в коде.

Протокол публикации:

1. Worker пишет в attempts/{attempt_id}/ и закрывает файлы. Сообщает относительные пути, размеры и хеши.
2. Laravel проверяет действующий token, входные версии и control. Создаёт durable receipt и переводит item в publishing; длительную проверку/копирование передаёт app-worker.
3. Повтор complete с тем же idempotency key возвращает тот же receipt. Изменённое тело с тем же ключом — 409.
4. App-worker копирует в свой staging, куда Python не может писать; сверяет хеш копии и формат.
5. Перед финальной публикацией ещё раз проверяет актуальность попытки, input fingerprint и отмену.
6. Создаёт уникальный immutable artifact path, атомарно переименовывая staging на том же filesystem. Не заменяет результат другой попытки.
7. В транзакции устанавливает ссылку Artifact, состояние succeeded, счётчики и событие.
8. Сбой между файловой операцией и БД оставляет восстанавливаемый/осиротевший файл. Reconciliation сверяет receipt, запись и хеш; не считает файл успешным только по наличию имени.
9. Старый worker не может затереть принятый файл, даже если продолжает писать в свой attempts-каталог.

Running-аренда Python не продлевается бесконечно для publishing: после durable receipt ответственность принимает Laravel. Для publishing задаются собственный deadline и повтор app-worker с тем же receipt.
Подробные сроки и протокол worker определены в CONTRACTS.md.

## D8. Корзина, очистка и backup

- Мягкое удаление скрывает элемент, сохраняя ссылки ревизий. В UI показаны срок и размер корзины.
- Физически удалять SourceFile/Artifact можно только после проверки отсутствия ссылок у активных коллекций, ревизий и выполняющихся задач.
- В первой версии восстановление из корзины возможно до явного безвозвратного удаления; автоматическое удаление пользовательских данных выключено.
- Временные файлы очищаются по возрасту и отсутствию активной попытки; предварительный просмотр очистки показывает объём.
- Время хранения доставленного outbox и технических логов задаётся отдельно от пользовательских ревизий.
- Backup включает PostgreSQL, библиотеку, .env/секреты и Laravel APP_KEY. Секретная часть хранится с ограниченными правами.
- Простой первый согласованный backup: режим обслуживания, пауза записи/worker, завершение публикаций, dump БД и копия файлов/секретов, manifest и хеши, затем возобновление.
- Restore сначала выполняется в отдельный пустой каталог/volume. Успех — файлы читаются, ревизии целостны, токены расшифровываются; рабочие данные не перезаписывать при пробной проверке.

## D9. Защита силуэта и происхождение генерации

Обязательно читать [R2–R4 дополнения](REQUIREMENTS_ADDENDUM.md).
Добавить immutable ProtectionRevision: asset_id, source_hash, normalized_width/height, mask_artifact_id, margin, model/version, confidence, manual_author, parent_revision_id.
TransformSpec ссылается на protection_revision_id и содержит scale, retained_rect, canvas, offset, generation_mask_hash, allowed_directions, generated_area_ratio и workflow_revision.
GenerationRun связывает attempt_id, external_id (nullable при неопределённой отправке), workflow_hash, model versions, seed, параметры и external_state.
ResultQA хранит artifact_hash, transform/protection revisions, pixel_check, geometry_check, semantic_findings и review_state.
ReviewDecision неизменяем: actor, verdict, reason, expected artifact/QA revisions, timestamp. Новый результат требует нового решения.
Все эти сущности включены в версии зависимостей preflight/manifest и правила D7 публикации; worker не пишет их напрямую.
Изменение маски поднимает версию draft, делает transform/QA/caption stale, не уничтожая ручной текст.
Collection.archived_at управляет архивом отдельно от deleted_at. Проверка владельца обязательна.
Состояния из требований UPLOADED/ANALYZING/PROCESSING и т.д. — отображаемые стадии конвейера, а не замена независимых статусов Job, пригодности, ревью и экспорта.

