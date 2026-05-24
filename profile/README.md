# Fedarisha — quickstart

Форк Remnawave с собственным транспортом `fedarisha` (PAK-токены, S3-доставка). Все образы опубликованы в Docker Hub и GHCR — собирать ничего не нужно.

| Сервис | Образ |
| --- | --- |
| xray-core | `voltara13/xray-core:latest` · `ghcr.io/fedarisha/xray-core:latest` |
| node | `voltara13/node:latest` · `ghcr.io/fedarisha/node:latest` |
| backend (+ frontend) | `voltara13/backend:latest` · `ghcr.io/fedarisha/backend:latest` |
| subscription-page | `voltara13/subscription-page:latest` · `ghcr.io/fedarisha/subscription-page:latest` |

Теги версий — `vX.Y.Z-fed.N` (`X.Y.Z` — апстрим, `-fed.N` — счётчик форка).

---

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

`.env` — стандартный remnawave (`POSTGRES_*`, `JWT_AUTH_SECRET`, `JWT_API_TOKENS_SECRET`, `WEBHOOK_*`, `SUB_PUBLIC_DOMAIN`, …; полный список в `backend/.env.sample`).

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
REMNAWAVE_PANEL_URL=http://remnawave:3000        # или https://panel.example.com если на другом хосте
REMNAWAVE_API_TOKEN=<токен из админки панели>
```

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
      - SECRET_KEY="..."
```

Node уже содержит `xray` + `geoip.dat`/`geosite.dat` из `voltara13/xray-core`.

## 4. Reverse-proxy (Caddy)

Subscription-page сам обрабатывает fedarisha-роуты (`fedarisha-json` зарегистрирован как client-type в `root.controller.ts`), отдельный rewrite не нужен:

```caddy
sub.example.com {
    reverse_proxy * http://remnawave-subscription-page:3010
}

panel.example.com {
    reverse_proxy http://remnawave:3000
}
```

## 5. Включение fedarisha в панели

1. `Settings → API Tokens` → создать токен, прописать в `subscription-page/.env` как `REMNAWAVE_API_TOKEN`.
2. В Xray-конфиге панели добавить inbound с `"protocol": "fedarisha"`. Валидатор требует webhook-блок (см. `backend/src/common/utils/apply-fedarisha-webhook-defaults.ts`) — туда подставляются S3-настройки доставки PAK-конфигов.
3. На каждой ноде указать `SECRET_KEY` (выдаётся панелью при регистрации) и `NODE_PORT`. Node поднимет xray + REST-сервис, принимающий `/node/fedarisha/{provision,revoke,probe}-user`.
4. Создать Internal Squad с fedarisha-inbound и привязать пользователей. Backend начнёт ходить в node `provision-user`, node — писать PAK в S3-бакет.
5. Subscription-URL для fedarisha-клиентов: `https://sub.example.com/{shortUuid}/fedarisha-json` — этот URL зашивается в клиент (v2rayN-core-bin / v2rayNG-форк).

---

# Сборка из исходников

Готовые образы покрывают 99 % сценариев. Сборка вручную нужна, если правите форк.

## Что клонировать

```bash
mkdir -p ~/fedarisha && cd ~/fedarisha
git clone https://github.com/Fedarisha/Xray-core-fedarisha  Xray-core
git clone https://github.com/Fedarisha/node                 node
git clone https://github.com/Fedarisha/backend              backend
git clone https://github.com/Fedarisha/frontend             frontend
git clone https://github.com/Fedarisha/subscription-page    subscription-page
```

Пин-файлы между юнитами:

- `node/.xray-core-version` → тег xray-core, против которого построен node;
- `backend/.frontend-version` → тег frontend, против которого построен backend.

Перед сборкой проверьте, что чекаут зависимости соответствует пину.

## Цепочка сборки

```
xray-core  →  node  →  backend+frontend  →  subscription-page
```

### Xray-core

Dockerfile лежит в `.github/docker/Dockerfile`. Альтернатива — готовые бинарники в релизах `Fedarisha/xray-builds` (`Xray-linux-64.zip`, `Xray-linux-arm64-v8a.zip`, Windows/macOS/Android).

```bash
cd ~/fedarisha/Xray-core
docker build -f .github/docker/Dockerfile \
  -t voltara13/xray-core:v26.5.3-fed.6 -t voltara13/xray-core:latest .
```

Локально (без Docker): `CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o ../bin/xray ./main` (Go ≥ 1.26, см. `go.mod`).

### Node

Подхватывает xray multi-stage из `XRAY_IMAGE` (по умолчанию `ghcr.io/fedarisha/xray-core:latest`).

```bash
cd ~/fedarisha/node
docker build \
  --build-arg XRAY_IMAGE=voltara13/xray-core:v26.5.3-fed.6 \
  -t voltara13/node:v26.5.3-fed.6 -t voltara13/node:latest .
```

В node живут модули fedarisha: `src/modules/fedarisha-pak/*` (`provision-user`, `revoke-user`, `probe-user`) и хендлер `handler.service.ts`, который добавляет/удаляет fedarisha-аккаунты в Xray.

### Frontend → zip

Backend забирает frontend как zip из релиза (`ARG FRONTEND_URL`). Соберите frontend и опубликуйте релиз с ассетом `remnawave-frontend.zip`:

```bash
cd ~/fedarisha/frontend
npm ci
npm run start:build
( cd dist && zip -r ../remnawave-frontend.zip . )
# затем gh release create vX.Y.Z-fed.N remnawave-frontend.zip
```

По умолчанию backend ходит на `https://github.com/Fedarisha/frontend/releases/latest/download/remnawave-frontend.zip`.

### Backend

```bash
cd ~/fedarisha/backend
docker build \
  -t voltara13/backend:v2.7.4-fed.2 -t voltara13/backend:latest .
```

Если frontend-репо приватный — пробросьте токен: `--secret id=clone_token,src=/tmp/clone_token`.

В backend живёт `src/modules/fedarisha-provisioning/*` и контракт node-вызовов `src/common/axios/fedarisha-node-contract.ts` (`/node/fedarisha/{provision,revoke,probe}-user`) — node должен уметь отвечать на эти ручки.

### Subscription-page

```bash
cd ~/fedarisha/subscription-page
( cd frontend && npm ci && npm run start:build )   # → frontend/dist/, который Dockerfile копирует
docker build -t voltara13/subscription-page:latest .
```

## Релизы

В каждом репо `.github/workflows/build-and-push.yml` собирает мульти-arch образ на push тега и публикует в Docker Hub (`voltara13/*`) и GHCR (`ghcr.io/fedarisha/*`). Нужны секреты `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `TOKEN_GH_DEPLOY`.

Порядок ручного релиза (та же схема тегирования):

1. **xray-core** → тег `vX.Y.Z-fed.N` (`X.Y.Z` из `core/core.go`).
2. **node** → обновить `.xray-core-version`, закоммитить, тег `vX.Y.Z-fed.N` (`X.Y.Z` из `package.json`).
3. **backend + frontend** → версии в обоих `package.json` совпадают; обновить `backend/.frontend-version`, закоммитить, один и тот же тег на оба репо.
4. **subscription-page** → независим, тегается отдельно.
