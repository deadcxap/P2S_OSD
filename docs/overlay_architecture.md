# Проект: OSD-оверлей для видеопотока Bambu Lab P2S

## Цель
Собрать единое контейнерное приложение, которое:
- принимает RTSP(S) поток камеры принтера;
- получает телеметрию/статус печати по MQTT (LAN-only + dev mode);
- накладывает OSD (статус/прогресс/температуры/AMS и т.д.) поверх видео;
- отдаёт готовый поток для ретрансляции (например, в OBS, go2rtc и др.);
- устойчиво работает при потере MQTT/видео и имеет healthcheck endpoint.

## Обязательные требования (как трактуем)
1. **Один контейнер**: все компоненты запускаются внутри одного образа.
2. **Без GPU**: software decode/encode (CPU-only).
3. **Автостарт**: контейнер стартует сам, без ручных команд.
4. **Наблюдаемость**:
   - структурированные логи (INFO/WARN/ERROR),
   - endpoint `/healthz` (liveness),
   - endpoint `/readyz` (readiness с деталями MQTT/RTSP/выходного потока).
5. **Отказоустойчивость**:
   - reconnect для RTSP и MQTT с backoff,
   - деградация (показывать заглушку/последнее состояние),
   - процесс не падает на разрывах сети.

---

## Вариант A (рекомендованный): Python orchestration + FFmpeg drawtext

### Идея
- Python-сервис получает MQTT-состояние (OpenBambuAPI схема), формирует текст OSD и пишет его в файл (`/run/osd.txt`).
- FFmpeg читает RTSP поток и накладывает `drawtext=textfile=/run/osd.txt:reload=1`.
- Выход публикуется как RTSP (локальный сервер внутри контейнера) или SRT/RTMP/MPEG-TS (в зависимости от потребителя).
- Встроенный HTTP-сервер Python отдаёт `/healthz` и `/readyz`.

### Инструменты
- `python:3.12-slim` + `paho-mqtt` + `fastapi`/`uvicorn`.
- `ffmpeg` (software x264/aac).
- (Опционально) встроенный `rtsp-simple-server`/`mediamtx` бинарь внутри контейнера, если нужен локальный RTSP output.

### Плюсы
- Быстрый старт и простая разработка.
- Гибкая бизнес-логика OSD (форматирование, локализация, fallback).
- Легко расширять (PNG-иконки, цветовая индикация, алерты).

### Минусы
- Текстовый OSD через drawtext сложнее для «богатого» интерфейса.
- Нужно аккуратно синхронизировать перезапуски FFmpeg при фатальных ошибках.

### План реализации
1. **MVP 1**:
   - MQTT клиент + parser основных полей (progress, layer, temps, job name, ETA).
   - writer `/run/osd.txt` атомарной заменой.
   - FFmpeg pipeline `rtsps -> overlay -> output`.
2. **MVP 2**:
   - retry/backoff и state machine источников.
   - `/healthz` (жив процесс), `/readyz` (подключение + возраст телеметрии).
3. **MVP 3**:
   - AMS блок (катушка: слот/тип/цвет), вентиляторы, расширенные статусы.
   - JSON-логи и метрики (счётчики reconnect).
4. **Hardening**:
   - ограничение CPU, watchdog FFmpeg subprocess,
   - таймауты, graceful shutdown, unit tests parser.

---

## Вариант B: GStreamer (единый media graph)

### Идея
- GStreamer pipeline принимает RTSP, поверх `textoverlay`/`cairooverlay` рисует OSD, затем отдаёт RTSP/RTMP.
- Python/Go sidecar-процесс внутри контейнера обновляет overlay-данные через shared memory/IPC.

### Инструменты
- `gstreamer1.0` + плагины `good/bad/ugly/libav`.
- Управляющий сервис на Python/Go.

### Плюсы
- Более «медийно-нативный» стек, гибкий роутинг потоков.
- Можно делать более сложную графику, чем drawtext.

### Минусы
- Существенно выше порог входа и отладка сложнее.
- Больше риск нестабильности на старте из-за плагинов/совместимости.

### План
1. Прототип pipeline с фиксированным overlay.
2. Канал обновления текста/графики из MQTT-сервиса.
3. Добавление reconnect и fallback веток.
4. Интеграция health/readiness + нагрузочные тесты.

---

## Вариант C: Go (основная логика) + FFmpeg subprocess

### Идея
- Основной сервис на Go:
  - MQTT ingestion,
  - хранение состояния,
  - HTTP health,
  - менеджер жизненного цикла FFmpeg.
- FFmpeg отвечает только за трансформацию медиа.

### Инструменты
- Go 1.23+, `eclipse/paho.mqtt.golang`, `zerolog`.
- FFmpeg.

### Плюсы
- Надёжный runtime, простой деплой статического бинаря.
- Удобно делать supervisor-логику, backoff и telemetry.

### Минусы
- Быстрее писать сложнее, чем на Python.
- Для красивого OSD всё равно ограничение drawtext (если не городить рендерер).

### План
1. Реализовать MQTT state store + форматтер OSD.
2. Процесс-менеджер FFmpeg с автоперезапуском.
3. Health/readiness с диагностикой источников.
4. Интеграционные тесты reconnect сценариев.

---

## Формат данных OSD (базовый)
Рекомендованный блок на экране:
- `Printer: {state}`
- `Job: {task_name}`
- `Progress: {percent}% | Layer: {cur}/{total}`
- `Elapsed: {hh:mm:ss} | ETA: {hh:mm:ss}`
- `Nozzle/Bed/Chamber: {t1}/{t2}/{t3}°C`
- `Fans: part={}% aux={}% chamber={}%`
- `AMS: slot={n} type={pla/petg/...} color=#{hex}`
- `Updated: {timestamp}`

Если MQTT неактивен:
- показывать `Telemetry: DISCONNECTED (last update N sec ago)`.

Если видео неактивно:
- выдавать сервисный поток-заглушку с текстом `VIDEO SOURCE DISCONNECTED` + продолжать публиковать health/readiness.

---

## Надёжность и эксплуатация

### Reconnect стратегия
- MQTT: экспоненциальный backoff 1s → 2s → 4s ... до 30s, jitter.
- RTSP: аналогично, с ограничением частоты рестартов FFmpeg.
- Circuit-breaker на «частые падения» с логом причины.

### Логи
- Формат JSON, поля:
  - `ts`, `level`, `component`, `event`, `printer_ip`, `details`.
- Ключевые события:
  - connect/disconnect MQTT,
  - start/stop FFmpeg,
  - stale telemetry,
  - output stream status.

### Health endpoints
- `/healthz`: 200 если процесс жив.
- `/readyz`: 200 когда есть рабочий output pipeline (даже в degraded mode можно 200 + флаг),
  иначе 503.
- `/metrics` (опционально): Prometheus counters/gauges.

---

## Предложение по структуре контейнера (для варианта A)

```text
/app
  main.py              # оркестратор
  mqtt_client.py       # подписка и нормализация статуса
  osd_formatter.py     # форматирование строк OSD
  ffmpeg_runner.py     # запуск/контроль ffmpeg
  health_api.py        # /healthz /readyz
  config.py            # env-конфиг
/run
  osd.txt              # текущее содержимое overlay
```

### Переменные окружения (минимум)
- `PRINTER_IP`
- `PRINTER_ACCESS_CODE`
- `PRINTER_SERIAL`
- `RTSP_PASSWORD`
- `MQTT_PORT` (обычно 8883/в зависимости от LAN-mode реализации)
- `OUTPUT_MODE` (`rtsp|rtmp|mpegts`)
- `OUTPUT_URL`
- `LOG_LEVEL`

---

## План работ (по спринтам)

### Спринт 1 (2-3 дня)
- Каркас контейнера, автозапуск, health endpoints.
- MQTT ingest + OSD txt.
- FFmpeg overlay + локальный output.

### Спринт 2 (2-4 дня)
- Расширенная телеметрия (AMS/вентиляторы/слои/ETA).
- Устойчивость к разрывам, backoff, watchdog.
- Улучшение логов и readiness-логики.

### Спринт 3 (2-3 дня)
- Нагрузочное тестирование CPU-only.
- Оптимизация параметров кодека и latency.
- Документация и примеры запуска.

---

## Рекомендация
Для начала выбрать **Вариант A (Python + FFmpeg)** как самый быстрый путь к рабочему MVP с минимальным риском. После валидации можно мигрировать управляющую часть в Go (Вариант C), если потребуется выше надёжность/производительность длительных аптаймов.
