# Quickstart

Развёртывание полного стека на готовых образах. По итогу получите:

- **Панель** на `panel.example.com` (backend + Postgres + Valkey).
- **Subscription-page** на `sub.example.com`, отдающую `fedarisha-json` для форк-клиентов.
- **Одну ноду** на отдельной машине, которая будет выдавать PAK-ключи на S3-бакет и проксировать через него трафик.
- **Один fedarisha-инбаунд** в xray-config панели, привязанный к Internal Squad.

Если стек уже знаком по upstream Remnawave — отличия только в образах (`voltara13/*`) и в одном новом блоке настроек инбаунда. Всё остальное — стандартный compose-флоу.

Параметры самого fedarisha-инбаунда (storage / tuning / webhook) — в [inbound-config.md](inbound-config.md); выбор S3-провайдера и формат блока `storage` — в [storage-providers.md](storage-providers.md).

## 1. Панель (backend + Postgres + Valkey)

`/opt/remnawave/docker-compose.yml`:

```yaml
x-common: &common
  ulimits:
    nofile:
      soft: 1048576
      hard: 1048576
  restart: always
  networks:
    - remnawave-network

x-logging: &logging
  logging:
    driver: json-file
    options:
      max-size: 100m
      max-file: 5

x-env: &env
  env_file: .env

services:
  remnawave:
    image: voltara13/backend:latest
    container_name: remnawave
    hostname: remnawave
    <<: [*common, *logging, *env]
    volumes:
      - valkey-socket:/var/run/valkey
    ports:
      - 127.0.0.1:3000:${APP_PORT:-3000}
      - 127.0.0.1:3001:${METRICS_PORT:-3001}
    healthcheck:
      test: ['CMD-SHELL', 'curl -f http://localhost:${METRICS_PORT:-3001}/health']
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s
    depends_on:
      remnawave-db:
        condition: service_healthy
      remnawave-redis:
        condition: service_healthy

  remnawave-db:
    image: postgres:17.6
    container_name: remnawave-db
    hostname: remnawave-db
    <<: [*common, *logging, *env]
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
      - TZ=UTC
    ports:
      - 127.0.0.1:6767:5432
    volumes:
      - remnawave-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}']
      interval: 3s
      timeout: 10s
      retries: 3

  remnawave-redis:
    image: valkey/valkey:9-alpine
    container_name: remnawave-redis
    hostname: remnawave-redis
    <<: [*common, *logging]
    volumes:
      - valkey-socket:/var/run/valkey
    command: >
      valkey-server
      --save ""
      --appendonly no
      --maxmemory-policy noeviction
      --loglevel warning
      --unixsocket /var/run/valkey/valkey.sock
      --unixsocketperm 777
      --port 0
    healthcheck:
      test: ['CMD', 'valkey-cli', '-s', '/var/run/valkey/valkey.sock', 'ping']
      interval: 3s
      timeout: 3s
      retries: 3

networks:
  remnawave-network:
    name: remnawave-network
    driver: bridge
    external: false

volumes:
  remnawave-db-data:
    name: remnawave-db-data
    driver: local
    external: false
  valkey-socket:
    name: valkey-socket
    driver: local
    external: false
```

`.env` — стандартный remnawave (`POSTGRES_*`, `JWT_AUTH_SECRET`, `JWT_API_TOKENS_SECRET`, `WEBHOOK_*`, `SUB_PUBLIC_DOMAIN`, …; полный список — в `backend/.env.sample`).

```bash
cd /opt/remnawave && docker compose pull && docker compose up -d
```

## 2. Subscription-page

`/opt/remnawave-subscription-page/docker-compose.yml`:

```yaml
services:
  remnawave-subscription-page:
    image: voltara13/subscription-page:latest
    container_name: remnawave-subscription-page
    hostname: remnawave-subscription-page
    restart: always
    env_file:
      - .env
    ports:
      - '127.0.0.1:3010:3010'
    networks:
      - remnawave-network

networks:
  remnawave-network:
    driver: bridge
    external: true
```

`.env`:

```
APP_PORT=3010
REMNAWAVE_PANEL_URL=http://remnawave:3000        # или https://panel.example.com, если на другом хосте
REMNAWAVE_API_TOKEN=<токен из админки панели>
```

Sub-page сам обрабатывает client-type `fedarisha-json` (зарегистрирован в `root.controller.ts`) — отдельный rewrite или middleware не нужен.

## 3. Node (на каждой выходной машине)

```yaml
services:
  remnanode:
    container_name: remnanode
    hostname: remnanode
    image: voltara13/node:latest
    network_mode: host
    restart: always
    cap_add:
      - NET_ADMIN
    ulimits:
      nofile:
        soft: 1048576
        hard: 1048576
    environment:
      - NODE_PORT=2222
      - SECRET_KEY="..."             # выдаётся панелью при регистрации ноды
```

Образ ноды уже содержит `xray` + `geoip.dat`/`geosite.dat` из `voltara13/xray-core` — отдельно качать ничего не нужно.

`network_mode: host` нужен, чтобы webhook-listener инбаунда (по умолчанию `:80`) был доступен из сети S3-провайдера. Если оборачиваете ноду в reverse-proxy — можно перейти на bridge и пробросить порт явно.

## 4. Reverse-proxy (Caddy)

```caddy
sub.example.com {
    reverse_proxy * http://remnawave-subscription-page:3010
}

panel.example.com {
    reverse_proxy http://remnawave:3000
}
```

Никаких отдельных rewrite'ов для `fedarisha-json` — sub-page сам отвечает на `/{shortUuid}/fedarisha-json` (см. [subscription-flow.md](subscription-flow.md#что-отдаёт-subscription-page)).

## 5. Включение fedarisha в панели

К этому моменту панель, sub-page и нода живые, но ни одного fedarisha-инбаунда ещё нет. Финальные шаги:

1. **API-токен для sub-page.** `Settings → API Tokens` → создать токен, прописать в `subscription-page/.env` как `REMNAWAVE_API_TOKEN`, перезапустить sub-page.
2. **Зарегистрировать ноду.** В UI панели завести node-record, скопировать `SECRET_KEY` в `.env` ноды.
3. **Добавить fedarisha-инбаунд.** В xray-конфиге config-profile панели добавить элемент с `"protocol": "fedarisha"`. Полная схема — [inbound-config.md](inbound-config.md), формат `settings.storage` — [storage-providers.md](storage-providers.md). Если задать `"webhook": {}` — backend подставит дефолты (`:80`, `/webhook`, `autoSetup: true`).
4. **Создать Internal Squad** с этим инбаундом и привязать пользователей. На каждый `USER.ENABLED` backend дёрнет `/node/fedarisha/provision-user` → нода выпишет PAK у S3-провайдера и добавит юзера в живой xray-runtime.
5. **Раздать ссылку.** Subscription-URL для fedarisha-клиентов: `https://sub.example.com/{shortUuid}/fedarisha-json` — этот URL зашивается в форк-клиенты ([v2rayN](https://github.com/Fedarisha/v2rayN), [v2rayNG](https://github.com/Fedarisha/v2rayNG)).

Дальше — обычный Remnawave-флоу: пользователи, squad'ы, квоты, метрики. Fedarisha не меняет ничего, кроме того, что один из инбаундов теперь S3-backed.
