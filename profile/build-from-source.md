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

Подхватывает xray multi-stage из `XRAY_IMAGE` (по умолчанию `voltara13/xray-core:latest`).

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
zip -r remnawave-frontend.zip dist          # архив ДОЛЖЕН содержать каталог dist/ внутри
# затем gh release create vX.Y.Z-fed.N remnawave-frontend.zip
```

> Внимание: backend's `Dockerfile` распаковывает архив в `frontend_temp/` и затем `COPY --from=frontend /opt/frontend/frontend_temp/dist ./frontend` — значит zip обязан содержать верхний каталог `dist/`. Если запаковать `( cd dist && zip -r ../x.zip . )` (без `dist/` в архиве), backend упадёт на следующем `curl -o frontend_temp/dist/assets/...` с `No such file or directory`.

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

В каждом репо `.github/workflows/build-and-push.yml` собирает мульти-arch образ на push тега и публикует в Docker Hub (`voltara13/*`). Нужны секреты `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`.

Порядок ручного релиза (та же схема тегирования):

1. **xray-core** → тег `vX.Y.Z-fed.N` (`X.Y.Z` из `core/core.go`).
2. **node** → обновить `.xray-core-version`, закоммитить, тег `vX.Y.Z-fed.N` (`X.Y.Z` из `package.json`).
3. **backend + frontend** → версии в обоих `package.json` совпадают; обновить `backend/.frontend-version`, закоммитить, один и тот же тег на оба репо.
4. **subscription-page** → независим, тегается отдельно.
