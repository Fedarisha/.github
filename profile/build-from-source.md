# Сборка и релизы

## Репозитории

```bash
git clone https://github.com/Fedarisha/Xray-core-fedarisha Xray-core
git clone https://github.com/Fedarisha/node
git clone https://github.com/Fedarisha/backend
git clone https://github.com/Fedarisha/frontend
git clone https://github.com/Fedarisha/subscription-page
```

Каждый форк имеет remote `upstream` на соответствующий репозиторий [XTLS](https://github.com/XTLS/Xray-core) или [remnawave](https://github.com/remnawave). Изменения апстрима вливаются через merge в `main`.

Теги: `<версия апстрима>-<версия Fedarisha>fed`, например `3.4.4-1.0.1fed`. У Xray-core тег начинается с `v`: `v26.9.9-1.0.1fed`. Версия апстрима своя в каждом репозитории (`core/core.go` для Xray, `package.json` для остальных), версия Fedarisha общая для всех репозиториев релиза.

## Зависимости между сборками

```
Xray-core-fedarisha (релиз с zip-архивами)  ──►  node (docker/Dockerfile, ARG XRAY_VERSION)
frontend (релиз с remnawave-frontend.zip)    ──►  backend (Dockerfile, ARG FRONTEND_URL; тег в .frontend-version)
subscription-page                               собирается независимо
```

## Xray-core-fedarisha

```bash
cd Xray-core
CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o xray ./main
```

Версия Go указана в `go.mod`. Код протокола находится в `proxy/fedarisha/`, парсер конфига — в `infra/conf/fedarisha.go`. Тесты: `go test ./proxy/fedarisha/... ./infra/conf/`.

Релиз: тег и GitHub Release с архивами `Xray-<платформа>.zip` (как у апстрима: `Xray-linux-64.zip`, `Xray-linux-arm64-v8a.zip`, `Xray-windows-64.zip`, …). Нода скачивает из релиза `Xray-linux-64.zip` и `Xray-linux-arm64-v8a.zip`. Те же архивы публикуются в [Fedarisha/xray-builds](https://github.com/Fedarisha/xray-builds), оттуда их берёт форк v2rayN.

## node

```bash
cd node
docker build -f docker/Dockerfile --build-arg XRAY_VERSION=v26.9.9-1.0.1fed -t fedarisha-node .
```

Версию Xray задаёт `ARG XRAY_VERSION` в `docker/Dockerfile`. Её значение по умолчанию — это версия, которая попадает в образ при сборке в CI. Файл `.xray-core-version` рядом нужен только для справки: CI его не читает. При обновлении ядра меняйте оба.

Для локальной разработки: `npm ci`, `npm run dev` (подробности в `DEV_ENV.md`).

## frontend

```bash
cd frontend
npm ci
npm run start:build
zip -qr remnawave-frontend.zip dist
```

Архив должен содержать каталог `dist/` в корне: Dockerfile backend распаковывает его и берёт `dist/`. При пуше тега CI (`release-frontend.yml`) собирает архив сам и прикладывает его к GitHub Release.

## backend

```bash
cd backend
docker build \
  --build-arg FRONTEND_URL=https://github.com/Fedarisha/frontend/releases/download/$(cat .frontend-version)/remnawave-frontend.zip \
  -t fedarisha-backend .
```

Без `FRONTEND_URL` берётся архив из последнего релиза frontend. Если репозиторий frontend приватный, передайте токен: `--secret id=clone_token,env=GH_TOKEN`.

## subscription-page

```bash
cd subscription-page
(cd frontend && npm ci && npm run start:build)
docker build -t fedarisha-subscription-page .
```

Dockerfile копирует уже собранный `frontend/dist/`, поэтому frontend нужно собрать до `docker build`.

## CI

В node, backend и subscription-page workflow `build-and-push.yml` запускается на пуш тега:

- собирает образы `linux/amd64` и `linux/arm64` и публикует мультиархитектурный образ `ghcr.io/fedarisha/<сервис>:<тег>` и `:latest`;
- если заданы секреты `DOCKERHUB_USERNAME` и `DOCKERHUB_TOKEN`, зеркалирует образ в Docker Hub как `voltara13/<сервис>`;
- создаёт GitHub Release.

Backend берёт frontend из релиза, указанного в `.frontend-version`.

## Порядок релиза

1. Xray-core-fedarisha: тег и релиз с архивами.
2. node: обновить `XRAY_VERSION` в `docker/Dockerfile` и `.xray-core-version`, закоммитить, поставить тег.
3. frontend: поставить тег (CI опубликует архив).
4. backend: обновить `.frontend-version`, закоммитить, поставить тег.
5. subscription-page: поставить тег при изменениях.
