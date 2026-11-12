# Рабочая среда — LDS-001

Измерено 12 ноября 2026. Предыдущая редакция сохранена в docs/archive/environment-before-s00.md.

| Параметр | Наблюдение |
| --- | --- |
| Windows | Windows 11 Pro, 10.0.26200; WSL сообщает build 26200.9711 |
| RAM Windows | 66 283 996 KiB видимой памяти, примерно 63.2 GiB |
| WSL | 2.5.9.0, kernel 6.6.87.2-microsoft-standard-WSL2 |
| Дистрибутив | Ubuntu 24.04.2 LTS, x86_64 |
| RAM WSL | total 7.8 GiB, available 5.4 GiB на момент снимка; swap 2 GiB |
| Docker CLI / daemon | 29.9.3 / 29.9.3; daemon отвечает |
| Compose | v5.7.0 |
| GPU | NVIDIA GeForce RTX 4070 Ti SUPER; 16376 MiB |
| Драйвер | nvidia-smi 615.78.05; Windows KMD 617.42 |
| CUDA в nvidia-smi | UMD 13.4: это НЕ установленный Toolkit и не проверка PyTorch |
| ФС WSL | /dev/sdf: 1007G, доступно 947G |
| Физический диск Windows C | 1.9T, доступно 253G; это ограничивает рост виртуального диска WSL |
| Git в WSL | 2.43.0 |

## Инвентаризация

src/ пуст. Приложение, compose.yaml, package/composer manifests отсутствуют.
Имеются PLAN.md, AUDIT.md, chat.md, LoRA_Dataset_Studio_Requirements.md,
docs/environment.md, docs/versions.md и docs/plan/*.md; .idea — локальные настройки IDE.
Пользовательская документация существует и сохраняется. AGENTS.md в проекте не обнаружен.
Git: исходный HEAD 8c42f58; до работы docs/versions.md имел staged пустой файл и рабочее содержимое.
Его рабочая редакция сохранена отдельно перед исправлением. src не удалялся и не переносился.

## Получение данных

PowerShell: Get-CimInstance Win32_OperatingSystem; wsl --version.
WSL: uname -a; cat /etc/os-release; free -h; df -h <project>; df -h /mnt/c;
docker --version; docker compose version; docker info --format '{{.ServerVersion}}';
nvidia-smi; nvidia-smi --query-gpu=name,memory.total --format=csv,noheader; git status --short.
Чтение WSL/CIM потребовало разрешённого запуска вне sandbox. Git используется внутри WSL;
глобальный safe.directory не изменялся.

## Ограничения

Доступная память и место — снимок, не резерв. WSL видит только 7.8 GiB из памяти хоста:
до ML-этапов измерить расход контейнеров и при необходимости явно согласовать увеличение лимита.
Версии и URL LM Studio/ComfyUI пока unknown; проверяются S07/S09a.
GPU passthrough в контейнер, CUDA Toolkit, PyTorch, vision и запуск приложения ещё не проверены.
