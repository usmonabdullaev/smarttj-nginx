# SmartTJ Nginx Reverse Proxy & SSL Gateway

Данный репозиторий содержит конфигурацию обратного прокси-сервера (Reverse Proxy) и терминатора SSL на базе **Nginx (Alpine)** в Docker для проекта **SmartTJ**.

Прокси-сервер выступает единой точкой входа (API Gateway / Ingress) для внешнего веб-трафика, перенаправляет HTTP на HTTPS, управляет SSL-сертификатами Let's Encrypt и проксирует запросы к внутренним сервисам через общую Docker-сеть.

---

## 📋 Содержание

- [Архитектура](#-архитектура)
- [Структура проекта](#-структура-проекта)
- [Маршрутизация и домены](#-маршрутизация-и-домены)
- [Требования](#-требования)
- [Быстрый старт и развертывание](#-быстрый-старт-и-развертывание)
  - [1. Создание общей Docker-сети](#1-создание-общей-docker-сети)
  - [2. Получение SSL-сертификатов Let's Encrypt](#2-получение-ssl-сертификатов-lets-encrypt)
  - [3. Запуск Nginx](#3-запуск-nginx)
- [Управление сервисом](#-управление-сервисом)
- [Особенности конфигураций](#-особенности-конфигураций)
- [Добавление нового сервиса](#-добавление-нового-сервиса)
- [Решение возможных проблем (Troubleshooting)](#-решение-возможных-проблем-troubleshooting)

---

## 🏗 Архитектура

```text
                               Клиент (Интернет)
                                       │
                                       ▼ (порты 80 / 443)
                         ┌───────────────────────────┐
                         │       smarttj-nginx       │
                         │      (nginx:alpine)       │
                         └─────────────┬─────────────┘
                                       │
                     Docker Network: smarttj-network (external)
                                       │
                ┌──────────────────────┼──────────────────────┐
                ▼                      ▼                      ▼
     ┌─────────────────────┐┌─────────────────────┐┌─────────────────────┐
     │   Backend Service   ││  Realtime Service   ││     n8n Service     │
     │    backend:3000     ││    realtime:3000    ││      n8n:5678       │
     │  (/api -> backend)  ││ (/realtime -> ws/rt)││    (все запросы)    │
     └─────────────────────┘└─────────────────────┘└─────────────────────┘
```

---

## 📁 Структура проекта

```text
smarttj-nginx/
├── docker-compose.yml       # Описание Docker-сервиса smarttj-nginx и монтируемых томов
├── nginx.conf               # Базовая конфигурация Nginx (events, mime.types, include conf.d/*.conf)
├── conf.d/                  # Каталог с конфигурациями виртуальных хостов
│   ├── default.conf         # Хост smarttj.duckdns.org (SSL, ACME, подключение conf.d/smarttj/*.conf)
│   ├── smarttj/             # Модульные маршруты для smarttj.duckdns.org
│   │   ├── api.conf         # Маршрутизация /api -> http://backend:3000
│   │   └── realtime.conf    # Маршрутизация /realtime -> http://realtime:3000 (WebSockets, SSE)
│   └── n8n.conf             # Маршрутизация smarttj-n8n.duckdns.org (n8n + WebSockets)
├── certbot/                 # Локальные директории для Certbot (игнорируются в git)
│   ├── conf/                # Конфигурации и сертификаты Let's Encrypt
│   └── www/                 # Веб-директория для проверки ACME (.well-known/acme-challenge)
├── .gitignore               # Исключения git (certbot/, .env*)
└── README.md                # Документация проекта
```

---

## 🌐 Маршрутизация и домены

| Домен                     | Маршрут (Path) | Внутренний адрес (Upstream) | Назначение                        | Конфиг-файл                    |
| ------------------------- | -------------- | --------------------------- | --------------------------------- | ------------------------------ |
| `smarttj.duckdns.org`     | `/api*`        | `http://backend:3000`       | REST API бэкенда                  | `conf.d/smarttj/api.conf`      |
| `smarttj.duckdns.org`     | `/realtime*`   | `http://realtime:3000`      | Realtime-сервис (WebSockets, SSE) | `conf.d/smarttj/realtime.conf` |
| `smarttj-n8n.duckdns.org` | `/*`           | `http://n8n:5678`           | Сервис автоматизации n8n          | `conf.d/n8n.conf`              |

Все хосты слушают порт `80` (HTTP) с редиректом `301 Moved Permanently` на `https://$host$request_uri`, за исключением маршрута ACME-челленджа `/.well-known/acme-challenge/`.

---

## ⚙️ Требования

- **Docker Engine** (20.10+)
- **Docker Compose** (v2+)
- Настроенные DNS-записи доменов (A-записи на внешний IP-адрес сервера)
- Открытые порты `80/tcp` и `443/tcp` на файрволе сервера

---

## 🚀 Быстрый старт и развертывание

### 1. Создание общей Docker-сети

Контейнер `smarttj-nginx` подключается к внешней сети `smarttj-network`. Если она ещё не создана на хосте, создайте её:

```bash
docker network create smarttj-network
```

> **Важно:** Все бэкенд-контейнеры (`backend`, `n8n` и др.) должны быть подключены к этой же сети `smarttj-network`, чтобы Nginx мог обращаться к ним по именам контейнеров.

---

### 2. Получение SSL-сертификатов Let's Encrypt

Сертификаты монтируются по путям:

- `/etc/letsencrypt` (системный каталог хоста)
- `./certbot/conf` и `./certbot/www`

#### Вариант А: Использование Certbot на хосте (Stand-alone или Webroot)

Если на хосте установлен `certbot`:

```bash
sudo certbot certonly --webroot -w ./certbot/www \
  -d smarttj.duckdns.org \
  -d smarttj-n8n.duckdns.org
```

Или standalone (если Nginx ещё не запущен и порт 80 свободен):

```bash
sudo certbot certonly --standalone \
  -d smarttj.duckdns.org \
  -d smarttj-n8n.duckdns.org
```

#### Вариант Б: Использование Certbot через Docker

```bash
docker run -it --rm --name certbot \
  -v "$(pwd)/certbot/conf:/etc/letsencrypt" \
  -v "$(pwd)/certbot/www:/var/www/certbot" \
  certbot/certbot certonly --webroot -w /var/www/certbot \
  -d smarttj.duckdns.org \
  -d smarttj-n8n.duckdns.org
```

---

### 3. Запуск Nginx

После того как сертификаты получены и находятся в `/etc/letsencrypt/live/`, запустите контейнер:

```bash
docker compose up -d
```

Проверьте статус контейнера:

```bash
docker compose ps
docker compose logs -f
```

---

## 🛠 Управление сервисом

- **Перезагрузить конфигурацию Nginx без простоя (Graceful Reload):**

  ```bash
  docker compose exec nginx nginx -s reload
  ```

- **Проверить корректность конфигурационных файлов:**

  ```bash
  docker compose exec nginx nginx -t
  ```

- **Перезапустить контейнер:**

  ```bash
  docker compose restart
  ```

- **Остановить контейнер:**

  ```bash
  docker compose down
  ```

- **Просмотр логов в реальном времени:**
  ```bash
  docker compose logs -f
  ```

---

## 🔍 Особенности конфигураций

1. **Динамический DNS-резолвер Docker (`resolver 127.0.0.11 valid=10s;` и `set $backend ...`)**:
   - По умолчанию Nginx при запуске резолвит доменные имена upstream один раз. Если контейнер `backend` в этот момент ещё не поднят, Nginx завершится с ошибкой `host not found`.
   - Использование директивы `resolver 127.0.0.11` (встроенный DNS Docker) в сочетании с переменными (например, `set $backend http://backend:3000; proxy_pass $backend;`) заставляет Nginx выполнять DNS-поиск динамически во время выполнения запроса.

2. **Конфигурация n8n (`conf.d/n8n.conf`)**:
   - **WebSockets**: Настроена передача заголовков `Upgrade` и `Connection "upgrade"`, необходимых для работы веб-интерфейса и редактора workflows n8n.
   - **Таймауты**: Установлены `proxy_read_timeout 300s;` и `proxy_send_timeout 300s;` для предотвращения обрыва долгих сценариев выполнения.
   - **Буферизация**: Отключена `proxy_buffering off;` для корректной передачи Server-Sent Events (SSE) и потоковых ответов.

3. **ACME Challenge**:
   - Во всех виртуальных хостах блок `location /.well-known/acme-challenge/` направлен в `/var/www/certbot`, что позволяет продлевать SSL-сертификаты без остановки Nginx.

---

## ➕ Добавление нового сервиса

Чтобы подключить новый сервис к проксированию:

1. Создайте новый конфигурационный файл в директории `conf.d/`, например `conf.d/my-service.conf`:

   ```nginx
   server {
       listen 80;
       server_name my-service.duckdns.org;

       location /.well-known/acme-challenge/ {
           root /var/www/certbot;
       }

       location / {
           return 301 https://$host$request_uri;
       }
   }

   server {
       listen 443 ssl;
       server_name my-service.duckdns.org;

       ssl_certificate /etc/letsencrypt/live/my-service.duckdns.org/fullchain.pem;
       ssl_certificate_key /etc/letsencrypt/live/my-service.duckdns.org/privkey.pem;

       set $upstream http://my-service-container:8080;

       location / {
           proxy_pass $upstream;

           proxy_http_version 1.1;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```

2. Убедитесь, что контейнер целевого сервиса подключен к сети `smarttj-network`.
3. Получите сертификат для нового домена через certbot.
4. Проверьте и примените конфигурацию:
   ```bash
   docker compose exec nginx nginx -t
   docker compose exec nginx nginx -s reload
   ```

---

## ❓ Решение возможных проблем (Troubleshooting)

### Ошибка: `network smarttj-network declared as external, but could not be found`

- **Причина:** Сеть Docker ещё не создана.
- **Решение:** Выполните `docker network create smarttj-network`.

### Ошибка: `cannot load certificate ... No such file or directory`

- **Причина:** Nginx пытается прочитать SSL-сертификаты, которые ещё не сгенерированы Let's Encrypt.
- **Решение:** Сначала выпустите сертификаты через Certbot (см. раздел [Получение SSL-сертификатов](#2-получение-ssl-сертификатов-lets-encrypt)).

### Ошибка: `502 Bad Gateway`

- **Причина:** Контейнер upstream (`backend` или `n8n`) не запущен, упал или не подключен к сети `smarttj-network`.
- **Решение:**
  1. Проверьте статус контейнеров: `docker ps`.
  2. Проверьте, подключен ли контейнер к сети:
     ```bash
     docker network inspect smarttj-network
     ```
  3. Проверьте логи целевого контейнера: `docker logs <container_name>`.
