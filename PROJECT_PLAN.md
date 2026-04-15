# Проект: оверлей телеметрии Bambu Lab P2S поверх RTSP-потока

## Цель
Собрать одно-контейнерное приложение, которое:
- получает видеопоток камеры принтера по `rtsps://...:322/streaming/live/1`;
- получает телеметрию/статус по MQTT (LAN-only + developer mode, совместимо с OpenBambuAPI);
- накладывает данные на видео в реальном времени;
- отдает готовый поток для OBS/go2rtc/другого ретранслятора;
- стабильно работает без GPU;
- имеет структурированные логи, healthcheck и устойчивость к пропаданию MQTT/RTSP.

---

## Базовые требования (чек-лист)
- [x] Один Docker-контейнер.
- [x] Автозапуск пайплайна при старте контейнера (без ручных команд).
- [x] Работа на CPU.
- [x] Читаемые логи состояния и ошибок.
- [x] Переживание обрывов MQTT и RTSP без падения процесса.
- [x] HTTP-эндпоинт health/readiness для мониторинга.
- [x] Возможность включить в контейнер сторонний проект (например, go2rtc), но в рамках одного контейнера.

---

## Вариант 1 (рекомендуемый): Python orchestration + FFmpeg drawtext + встроенный RTSP-сервер (MediaMTX)

### Идея
- Python-процесс:
  - подписывается на MQTT, нормализует телеметрию;
  - пишет текущий оверлей в текстовый файл (`/run/overlay.txt`);
  - держит HTTP `/healthz` и `/readyz`;
  - логирует состояние/ошибки (JSON-лог).
- FFmpeg-процесс:
  - читает RTSP;
  - накладывает `drawtext=textfile=/run/overlay.txt:reload=1`;
  - публикует поток в локальный MediaMTX (`rtsp://127.0.0.1:8554/p2s_overlay`).
- MediaMTX в том же контейнере раздает RTSP/RTMP/WebRTC (по необходимости).

### Плюсы
- Минимум собственного низкоуровневого видео-кода.
- Устойчивость: FFmpeg и MQTT можно рестартовать независимо (через supervisor).
- Простое масштабирование форматирования оверлея.

### Риски/минусы
- `drawtext` ограничен по сложной верстке (таблицы/иконки сложнее).
- Нужно аккуратно синхронизировать обновление файла оверлея (atomic write).

### План реализации
1. **Слой конфигурации**
   - ENV: IP принтера, пароль, MQTT-топики, формат выдачи, fps/битрейт.
2. **Сбор телеметрии**
   - MQTT client (paho-mqtt/asyncio-mqtt), reconnect с backoff.
   - Парсер payload по схеме OpenBambuAPI.
3. **Модель состояния**
   - Единый `PrinterState` с timestamp последнего обновления.
   - Поля: AMS (слот/тип/цвет), статус, прогресс, слой, job name, elapsed/remaining, температуры, вентиляторы, ошибки.
4. **Рендер оверлея в текст**
   - Шаблон строк (несколько строк с фиксированным порядком).
   - Атомарная запись через tmp + rename.
5. **Видео-пайплайн**
   - FFmpeg командой из env (preset, reconnect options).
   - В случае потери видео: автоreconnect без завершения контейнера.
6. **Выдача потока**
   - Поднять MediaMTX в том же контейнере, публиковать туда выход FFmpeg.
7. **Наблюдаемость**
   - `/healthz`: жив ли процесс.
   - `/readyz`: есть ли свежая телеметрия и/или видео за N секунд.
   - `/metrics` (опционально, Prometheus).
8. **Fail-safe поведение**
   - Нет MQTT: показывать `MQTT: disconnected`, последнее известное состояние.
   - Нет RTSP: показывать standby/заглушку либо держать процесс с reconnect.
9. **Docker**
   - multi-stage build, tini/s6/supervisord как init.
   - Один `CMD`, стартующий supervisor, который поднимает все процессы.

---

## Вариант 2: GStreamer (единый мультимедийный граф) + Python для телеметрии

### Идея
- GStreamer получает RTSP (`rtspsrc`), декодирует CPU (`avdec_h264`), накладывает текст (`textoverlay`/`cairooverlay`), кодирует (`x264enc`), отдает RTSP/UDP.
- Python процесс обновляет shared-state (например, через локальный сокет/redis-inproc-файл).

### Плюсы
- Гибче компоновка пайплайна и ниже latency при правильной настройке.
- Возможны более сложные оверлеи (через cairo).

### Риски/минусы
- Более крутая кривая настройки.
- Сложнее сопровождение для команды без опыта GStreamer.

### План реализации
1. Собрать baseline pipeline с reconnect.
2. Подключить динамический text source.
3. Реализовать health probes + watchdog pipeline.
4. Обернуть в supervisor и Docker.

---

## Вариант 3: go2rtc как ядро ретрансляции + sidecar-процесс оверлея внутри того же контейнера

### Идея
- go2rtc забирает исходный поток и отдает его локально.
- Отдельный процесс внутри контейнера (FFmpeg/Python) берет поток из go2rtc, добавляет overlay, публикует обратно вторым потоком.

### Плюсы
- Удобная интеграция, если у вас уже экосистема на go2rtc.
- Гибкая маршрутизация клиентов.

### Риски/минусы
- Чуть сложнее маршрутизация потоков и конфиги.
- Потребление CPU выше при двойной обработке.

### План реализации
1. Встроить бинарь go2rtc в образ.
2. Поднять stream source и overlay stream.
3. Прописать стабильные имена потоков и healthchecks.

---

## Рекомендованная структура проекта

- `app/main.py` — orchestration, lifecycle.
- `app/mqtt_client.py` — подключение/подписки/reconnect.
- `app/state.py` — модель состояния и нормализация.
- `app/overlay_renderer.py` — генерация `/run/overlay.txt`.
- `app/health_api.py` — `/healthz`, `/readyz`, `/metrics`.
- `docker/ffmpeg.sh` — запуск ffmpeg с reconnect-флагами.
- `docker/mediamtx.yml` — конфиг локальной раздачи потоков.
- `docker/supervisord.conf` — процессы: python, ffmpeg, mediamtx.

---

## Формат оверлея (пример)

```text
P2S | Printing | 42% | Layer 135/320
Job: gearbox_v7.3mf
AMS: Slot 2 | PLA Basic | #FF7A00
Elapsed: 01:12:33 | Remaining: 00:35:20
Nozzle: 218°C | Bed: 60°C | Chamber: 36°C
Fans: Part 70% | Aux 40% | Chamber 35%
MQTT: OK (0.8s) | Video: OK (0.2s)
```

---

## Отказоустойчивость (обязательная логика)

1. **MQTT down**
   - Не падать.
   - Статус в overlay: `MQTT disconnected`.
   - Экспоненциальный reconnect (до max interval).
2. **RTSP down**
   - Не падать.
   - FFmpeg reconnect (`-reconnect 1 -reconnect_streamed 1 -reconnect_delay_max 10`).
3. **Оба канала down**
   - Контейнер жив, `/healthz = 200`, `/readyz = 503`.
4. **Переполнение/битый payload**
   - Валидация входа, лог ошибки, продолжение работы.

---

## Логирование

- Формат: JSON Lines.
- Поля: `ts`, `level`, `component`, `event`, `message`, `error`, `printer_ip`.
- Важные события:
  - connect/disconnect/reconnect (mqtt, rtsp);
  - смена статуса печати;
  - смена задания;
  - деградация readiness.

---

## Health endpoints

- `GET /healthz` — процесс жив (всегда 200, пока event loop работает).
- `GET /readyz` — готовность сервиса:
  - 200: есть видео и/или телеметрия свежее N сек;
  - 503: оба источника stale.
- `GET /status` — краткий JSON со state (для дебага).

---

## Docker-стратегия

- База: `python:3.12-slim` + `ffmpeg` + `mediamtx` (скачанный бинарь).
- PID 1: `tini`.
- Процесс-менеджмент: `supervisord`/`s6-overlay`.
- `HEALTHCHECK` на `curl -f http://127.0.0.1:8080/healthz`.
- Автостарт через `CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]`.

---

## Этапы внедрения (MVP -> production)

1. **MVP (1–2 дня)**
   - MQTT ingest + простой overlay.txt + FFmpeg drawtext + `/healthz`.
2. **Стабилизация (2–4 дня)**
   - reconnect стратегия, `/readyz`, структурные логи, docker healthcheck.
3. **Production hardening (3–5 дней)**
   - watchdog процессов, graceful shutdown, ограничения CPU/RAM, документация.
4. **Опционально**
   - Preset-ы оверлея (минимальный/расширенный), локализация, экспорт метрик.

---

## Что выбрать сейчас

Для старта рекомендован **Вариант 1** как самый быстрый и предсказуемый для CPU-only контейнера.
Он лучше всего закрывает требования по надежности, простоте сопровождения и интеграции с go2rtc/OBS.
