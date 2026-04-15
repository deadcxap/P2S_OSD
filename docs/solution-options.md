# Проект: OSD-оверлей для видеопотока камеры Bambu Lab P2S

## Цель
Собрать сервис в **одном Docker-контейнере**, который:
- читает RTSP(S)-поток камеры принтера;
- получает телеметрию/статус печати через MQTT (LAN-only + developer mode);
- накладывает OSD поверх видео в реальном времени;
- отдаёт новый поток (RTSP/RTMP/HLS/WebRTC через ретранслятор);
- устойчив к обрывам видео и MQTT;
- имеет healthcheck endpoint и читаемые логи.

---

## Вариант 1 (рекомендуемый): Python оркестратор + FFmpeg (CPU) + go2rtc (опционально внутри контейнера)

### Идея
- Python-процесс:
  - подключается к MQTT принтера;
  - нормализует состояние печати;
  - пишет текст/JSON для оверлея в файл (atomically replace);
  - поднимает `/healthz` и `/metrics`.
- FFmpeg-процесс:
  - читает `rtsps://...`;
  - применяет `drawtext`/`overlay` (через textfile или PNG, генерируемый Python);
  - публикует в локальный RTSP/RTMP endpoint.
- go2rtc (одним бинарником в том же контейнере) при необходимости:
  - принимает поток от FFmpeg и раздаёт в нужные протоколы.

### Почему это хороший старт
- Минимум сложной графики, всё на CPU.
- Быстрый time-to-first-frame.
- Прозрачная диагностика через stdout/stderr FFmpeg и Python.
- Простое восстановление: перезапуск дочерних процессов и backoff.

### План внедрения
1. **Data layer**
   - MQTT client с автоreconnect и last-known-state.
   - Маппинг полей OpenBambuAPI → внутренний `PrinterState`.
2. **OSD formatter**
   - Формирование экранных строк: AMS, прогресс, слой, температуры, вентиляторы, ETA и т.д.
   - Вывод в `overlay.txt` + timestamp обновления.
3. **Video pipeline**
   - FFmpeg команда с `-reconnect` параметрами и `drawtext=textfile=...:reload=1`.
   - Выход в `rtsp://127.0.0.1:8554/p2s_osd` (или в go2rtc input).
4. **Resilience**
   - Supervisor-loop: если FFmpeg/ MQTT упал — перезапуск с экспоненциальной паузой.
   - Если MQTT недоступен: показывать `MQTT: disconnected`, но видео продолжать.
   - Если RTSP недоступен: держать сервис живым, отдавать health=degraded.
5. **Observability**
   - `/healthz` (ok/degraded/down + детали subsystems).
   - `/readyz` (stream up + mqtt optional/required флагом).
   - структурированные логи (JSON или key=value).

---

## Вариант 2: GStreamer (единый media pipeline) + Python контроллер

### Идея
- GStreamer как основной движок ingest/overlay/output.
- Оверлей через `textoverlay`/`cairooverlay` (динамика из Python через bus/shared state).

### Плюсы
- Гибкая мультимедиа-маршрутизация, хорош для сложных пайплайнов.
- Легче расширять на мультистримы и разные кодеки.

### Минусы
- Выше порог поддержки и дебага.
- Больше зависимостей в контейнере.

### План внедрения
1. Собрать базовый pipeline ingest→overlay→rtsp/rtmp.
2. Подключить динамическое обновление текста.
3. Реализовать watchdog и health endpoint.
4. Добавить профили качества/битрейта.

---

## Вариант 3: Веб-рендер OSD (HTML/CSS) + headless compositor + FFmpeg

### Идея
- Формировать OSD как «красивую» веб-плашку (HTML/CSS/JS), рендерить в PNG/WebM слой.
- FFmpeg накладывает слой на RTSP.

### Плюсы
- Максимально гибкий UI/брендинг.

### Минусы
- Выше потребление CPU/RAM.
- Больше точек отказа.

### Когда выбирать
- Если требуется продакшн-графика уровня трансляций (иконки, анимации, темы).

---

## Что собирать в первую очередь (MVP)

1. **Взять вариант 1** как самый быстрый и устойчивый.
2. Функционально закрыть обязательные поля OSD:
   - AMS: слот/тип/цвет;
   - статус принтера;
   - прогресс, слой `current/total`;
   - имя задания;
   - elapsed/remaining;
   - температуры сопла/стола/камеры;
   - скорости вентиляторов.
3. Добавить «деградацию без падения»:
   - нет MQTT → выводить `N/A`, поток не останавливать;
   - нет RTSP → процесс жив, health=degraded, автоreconnect.
4. Поднять внутренний endpoint мониторинга.

---

## Технический скелет контейнера

- `python:3.12-slim` + `ffmpeg` + (опционально) `go2rtc` binary.
- Один `ENTRYPOINT`, который запускает supervisor (Python) без дополнительных команд.
- `HEALTHCHECK` в Dockerfile на `/healthz`.
- Конфиг через env:
  - `PRINTER_IP`, `ACCESS_CODE`, `SERIAL`, `MQTT_PORT`, `RTSP_PASSWORD`;
  - `OUTPUT_MODE=rtsp|rtmp|hls`; `TZ`; `LOG_LEVEL`.

---

## Формат health endpoint

```json
{
  "status": "degraded",
  "time": "2026-04-15T12:00:00Z",
  "uptime_sec": 1234,
  "mqtt": {"connected": false, "last_msg_sec": 78},
  "video_in": {"connected": true, "fps": 14.8},
  "video_out": {"connected": true, "target": "rtsp://.../p2s_osd"},
  "state_age_sec": 9
}
```

---

## Риски и меры

- **Изменение формата MQTT-полей** → слой адаптера + версия схемы.
- **Нестабильный RTSP(S)** → conservative буфер, reconnect, ограничение latency.
- **Рост задержки** → фиксировать пресет кодека (`veryfast`/`superfast`), контролировать bitrate/GOP.

---

## Рекомендация

Начать с **Варианта 1** и за 2–3 итерации довести до рабочего контейнера с устойчивостью и healthcheck. После стабилизации можно перейти к улучшенному визуалу (вариант 3) без ломки data-layer.
