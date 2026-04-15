# P2S_OSD

Проект наложения телеметрии печати Bambu Lab P2S поверх RTSP-видеопотока камеры принтера.

## Цель

Получить единый выходной видеопоток (для OBS/go2rtc/других клиентов), в котором уже «вклеены»:

- состояние принтера;
- данные задания печати;
- текущий слой и прогресс;
- параметры AMS (катушка, тип, цвет, слот);
- температуры (сопло, стол, камера);
- скорости вентиляторов и иные доступные метрики.

Ограничения:

- принтер в режиме `LAN-only` + включённый `Developer Mode`;
- взаимодействие по MQTT (как в OpenBambuAPI);
- входное видео по `rtsps://...:322/streaming/live/1`;
- работа **без GPU** (CPU-only);
- один Docker-контейнер;
- автозапуск без ручных команд после старта контейнера;
- читаемые логи;
- устойчивость при пропаже MQTT/RTSP;
- endpoint healthcheck для мониторинга.

---

## Вариант 1 (рекомендуется): Python + GStreamer + Cairo/Pango OSD

### Идея

Собрать сервис на Python, который:

1. читает RTSP через GStreamer;
2. рисует OSD-слой на кадре в пайплайне (`cairooverlay`/`textoverlay`);
3. держит отдельный MQTT-клиент для статуса принтера;
4. публикует выход как RTSP (через встроенный `gst-rtsp-server`) или SRT/MPEG-TS;
5. поднимает HTTP endpoint `/healthz` и `/metrics`.

### Инструменты

- `python:3.12-slim`;
- `gstreamer1.0` + `-plugins-base/-good/-bad/-ugly`;
- Python-библиотеки: `paho-mqtt`, `fastapi`/`aiohttp`, `uvicorn`, `pydantic`;
- `gst-rtsp-server` (через C-плагин в runtime, либо запуск отдельного процесса `gst-rtsp-launch`, но внутри того же контейнера).

### Плюсы

- нативный real-time видеопайплайн;
- хорошая отказоустойчивость за счёт рестартуемых веток пайплайна;
- OSD рендерится без полного декод/энкод в Python-коде;
- удобная интеграция метрик и health endpoint.

### Минусы

- сложнее в настройке GStreamer;
- нужно аккуратно подобрать кодек/битрейт CPU-only.

### План реализации

1. **Каркас приложения**
   - `app/main.py` (оркестратор), `app/config.py`, `app/logging.py`.
2. **MQTT-адаптер**
   - подключение к топикам Bambu;
   - нормализация в единый `PrinterState`.
3. **State Store**
   - thread-safe/async-safe store;
   - TTL-поля (чтобы помечать устаревшие данные как `N/A`).
4. **Видеопайплайн**
   - RTSP ingest → decode → overlay → encode (`x264enc ultrafast`) → RTSP output.
5. **OSD layout**
   - несколько блоков: job, printer, temps, fans, AMS;
   - fallback-текст при отсутствии данных.
6. **Resilience**
   - retry с backoff для MQTT и RTSP;
   - watchdog пайплайна, автоматический re-init.
7. **Observability**
   - структурные логи JSON/текст;
   - `/healthz` (liveness/readiness), `/metrics`.
8. **Docker**
   - `ENTRYPOINT` запускает единый процесс-супервизор.

---

## Вариант 2: Go + FFmpeg filtergraph (`drawtext`/`overlay`) + MQTT

### Идея

Go-сервис получает MQTT-статус, генерирует текстовый OSD (или bitmap), а FFmpeg-процесс выполняет транскод и наложение через фильтры.

### Инструменты

- Go 1.23+;
- `ffmpeg` (libx264, drawtext);
- MQTT-клиент (`eclipse/paho.mqtt.golang`);
- HTTP сервер (`chi`/`net/http`) для health.

### Плюсы

- проще найти специалистов по FFmpeg;
- один стабильный бинарь-оркестратор;
- предсказуемое поведение в Docker.

### Минусы

- `drawtext` обновляется через перезапись textfile/zmq-команды, что усложняет динамику;
- сложнее сделать богатый UI (цветные блоки, иконки) без дополнительных костылей.

### План реализации

1. Go-daemon с подсистемами `mqtt`, `state`, `ffmpeg`, `health`.
2. Генерация `osd.txt` (atomic write) каждые 250–500 мс.
3. Запуск FFmpeg с `-vf drawtext=textfile=...:reload=1`.
4. Выходной поток: RTSP push в локальный go2rtc (внутри контейнера) или прямой RTMP/SRT.
5. Контроль процесса FFmpeg (перезапуск + throttling).
6. Prometheus/health endpoints.

---

## Вариант 3: Встроенный go2rtc + sidecar-процесс OSD внутри одного контейнера

### Идея

В одном контейнере работают:

- `go2rtc` как универсальный ретранслятор;
- отдельный OSD-процесс (Python/Go), который берёт исходный поток и публикует «обогащённый» поток обратно в go2rtc.

> Несмотря на два процесса, это **один контейнер**, что соответствует требованию.

### Плюсы

- сразу готовая интеграция с экосистемой go2rtc;
- удобно раздавать в разные протоколы (WebRTC/RTSP/HLS).

### Минусы

- нужно корректно организовать процесс-менеджмент (s6/supervisord/tini + watchdog);
- сложнее отладка при циклической маршрутизации потоков.

### План реализации

1. Собрать image с `go2rtc` + OSD service.
2. Статический `go2rtc.yaml` с входным потоком принтера и выходом `stream_osd`.
3. OSD service публикует в локальный ingest (`rtsp://127.0.0.1:8554/...`).
4. Единый `/healthz` агрегирует состояние обоих процессов.
5. Логи обоих процессов в stdout/stderr контейнера.

---

## Рекомендуемая архитектура MVP

Для первого релиза: **Вариант 1**.

Почему:

- лучший баланс между гибкостью OSD и стабильностью в real-time;
- можно начать с простого текстового overlay и постепенно добавить графические элементы;
- проще контролировать reconnect-логику в одном коде.

### MVP scope (итерация 1)

- вход RTSP;
- MQTT-подписка на базовые поля (status/progress/layers/temps);
- простое OSD (2 колонки);
- выход RTSP;
- `/healthz` и `/readyz`;
- Docker + `HEALTHCHECK`;
- устойчивый reconnect.

### Итерция 2

- AMS-детализация (тип/цвет/слот);
- красивые плашки/цвета;
- экспорт метрик Prometheus;
- профили качества (low/medium/high CPU).

---

## Структура репозитория (предложение)

```text
.
├─ app/
│  ├─ main.py
│  ├─ config.py
│  ├─ logging.py
│  ├─ state/
│  │  ├─ models.py
│  │  └─ store.py
│  ├─ mqtt/
│  │  ├─ client.py
│  │  └─ parser.py
│  ├─ video/
│  │  ├─ pipeline.py
│  │  └─ overlay.py
│  └─ api/
│     └─ health.py
├─ Dockerfile
├─ docker-compose.example.yml
├─ .env.example
└─ README.md
```

---

## Ключевые нефункциональные требования и как их закрыть

1. **Не падать при отсутствии коннекта**
   - бесконечный reconnect c exponential backoff;
   - «деградированный режим» (OSD показывает `NO MQTT` / `NO VIDEO`).
2. **Читаемые логи**
   - уровни `INFO/WARN/ERROR`, correlation-id сессии подключения;
   - отдельные события `mqtt_connected`, `rtsp_disconnected`, `pipeline_restarted`.
3. **Автозапуск**
   - `ENTRYPOINT ["python", "-m", "app.main"]`;
   - никаких ручных post-start команд.
4. **Проверка работоспособности**
   - `/healthz` (жив ли процесс);
   - `/readyz` (есть ли свежие state/video за N сек).
5. **CPU-only**
   - `x264enc tune=zerolatency speed-preset=ultrafast`;
   - ограничить fps/разрешение в конфиге.

---

## Что подготовить перед кодированием

- список точных MQTT topic/payload полей (по OpenBambuAPI и фактическим payload вашего принтера);
- целевой формат выходного потока (RTSP/RTMP/WebRTC);
- допустимая задержка (например, ≤ 2–3 с);
- CPU-бюджет хоста (число vCPU).

После этого можно переходить к реализации MVP в этом репозитории.
