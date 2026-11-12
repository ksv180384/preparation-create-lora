# Архитектура S00

Vue → Nginx → Laravel → PostgreSQL.
Python получает ProcessingJobItem через internal HTTP, пишет только attempts и сообщает Laravel.
Laravel публикует immutable artifacts и transactional outbox → Fastify → Socket.IO → Vue.
LM Studio и ComfyUI — внешние локальные AI через адаптер Python, общий GPU-slot.
Файлы — локальное хранилище WSL; в БД метаданные и версии.
Единственный публичный listener: 127.0.0.1:7676. Внутренние маршруты не открывать браузеру.
Сбой realtime не отменяет HTTP-сохранение; восстановление через snapshot.
Подробные схемы: ../plan/DATA_MODEL.md и ../plan/CONTRACTS.md.
