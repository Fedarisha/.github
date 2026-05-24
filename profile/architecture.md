# Архитектура форка

Что добавлено поверх Remnawave и как компоненты разговаривают между собой.

## Карта компонентов

```
                ┌─────────────────────────────────────────────────────┐
                │                  Panel (backend)                    │
                │                                                     │
                │  ┌───────────────────────────────────────────────┐  │
                │  │ fedarisha-provisioning module                 │  │
                │  │ — слушает USER.*  → ensureForUser/revokeForUser│  │
                │  │ — кеширует PAK в UserMeta.metadata.fedarisha  │  │
                │  │ — оркестратор HTTP-вызовов к нодам            │  │
                │  └───────────────────────────────────────────────┘  │
                │  ┌───────────────────────────────────────────────┐  │
                │  │ fedarisha-subscription.service                │  │
                │  │ — собирает per-user outbound для xray-json    │  │
                │  └───────────────────────────────────────────────┘  │
                └───────────────────────┬─────────────────────────────┘
                                        │
                          HTTPS, NODE_JWT (bearer)
                                        │
                ┌───────────────────────▼─────────────────────────────┐
                │                       Node                          │
                │                                                     │
                │  REST  /node/fedarisha/{provision,revoke,probe}-user│
                │  ┌───────────────────────────────────────────────┐  │
                │  │ FedarishaPakService (orchestrator)            │  │
                │  │   ├─ PakService          (vkcloud-pak)        │  │
                │  │   ├─ SelectelPakService  (selectel-iam)       │  │
                │  │   └─ StaticPakService    (static)             │  │
                │  └───────────────┬───────────────────────────────┘  │
                │                  │ S3 / IAM API                     │
                │                  ▼                                  │
                │  ┌───────────────────────────────────────────────┐  │
                │  │ xray-core (fedarisha inbound)                 │  │
                │  │  webhook listener  ←──── S3 ObjectCreated     │  │
                │  └───────────────────────────────────────────────┘  │
                └───────────────────────┬─────────────────────────────┘
                                        │
                          S3 API (PUT/GET/LIST/DELETE)
                                        │
                ┌───────────────────────▼─────────────────────────────┐
                │             Object storage (VK Cloud / Selectel /   │
                │             MinIO / Garage / любой S3-compat)       │
                │                                                     │
                │  <bucket>/<prefix>/<userId>/sessions/<sessionId>/   │
                │      c_00000000  s_00000000  c_00000001  ...        │
                └───────────────────────▲─────────────────────────────┘
                                        │
                          S3 API (PUT/GET/LIST/DELETE)
                                        │
                ┌───────────────────────┴─────────────────────────────┐
                │   Client (xray-core-bin / v2rayN-fork / v2rayNG-    │
                │   fork) — подписка отдаёт outbound с per-user PAK,  │
                │   клиент общается с node через S3, без TCP к ноде   │
                └─────────────────────────────────────────────────────┘
```

Между клиентом и нодой **нет сетевого соединения**: оба пишут/читают одни и те же объекты в одном бакете. PAK-ключи изолируют пользователей по префиксу.

## Что лежит в каждом репозитории

| Репо | Что добавлено форком | Главные пути |
| --- | --- | --- |
| [`Xray-core-fedarisha`](https://github.com/Fedarisha/Xray-core-fedarisha) | Transport `fedarisha`: парсер, listener/dialer, S3 store, webhook registry, X25519+AES-GCM crypto | `proxy/fedarisha/` (~5500 LOC), `infra/conf/fedarisha.go` |
| [`node`](https://github.com/Fedarisha/node) | NestJS-модуль `FedarishaPakModule`: REST-контракт + три провайдера PAK | `src/modules/fedarisha-pak/` (~900 LOC) |
| [`backend`](https://github.com/Fedarisha/backend) | `FedarishaProvisioningModule` (события + кеш UserMeta), `FedarishaSubscriptionService` (рендер outbound), webhook-дефолты, Zod-схема | `src/modules/fedarisha-provisioning/`, `src/common/utils/apply-fedarisha-webhook-defaults.ts`, `libs/contract/models/resolved-proxy-config.schema.ts` |
| [`subscription-page`](https://github.com/Fedarisha/subscription-page) | Регистрация client-type `fedarisha-json` | `backend/src/modules/root/root.controller.ts` |
| [`v2rayN`](https://github.com/Fedarisha/v2rayN), [`v2rayNG`](https://github.com/Fedarisha/v2rayNG) | Форки клиентов с поддержкой `fedarisha-json` и встроенным xray-core-fedarisha | импортируют core/xray-core как git submodule |

## Жизненный цикл одного пользователя

1. **Админ создаёт пользователя в панели и привязывает к Internal Squad с fedarisha-inbound.**
2. Backend получает `USER.CREATED` → `USER.ENABLED` через `event-emitter`. Если `status === ACTIVE`, `FedarishaProvisioningEvents.onUserEnabled` зовёт `ensureForUser(userId)`.
3. `ensureForUser` идёт в `internal_squad_inbounds`, находит все fedarisha-inbound'ы пользователя. Для каждого:
   - Резолвит ноду по `configProfileUuid`.
   - Шлёт `POST /node/fedarisha/provision-user` с `{userUuid, inboundTag, prefix}` (prefix = `<basePrefix>/<userId>/`).
4. Node `FedarishaPakService` смотрит на `inbound.settings.storage.type`, выбирает провайдера, выдаёт PAK:
   - `vkcloud-pak` — один S3-вызов `?prefixAccess`;
   - `selectel-iam` — токен Keystone → IAM service user → bucket policy upsert → credential;
   - `static` — pass-through мастер-ключей.
5. Backend сохраняет ответ в `user_meta.metadata.fedarisha[inboundTag] = {accessKey, secretKey, prefix, configProfileUuid, issuedAt}`.
6. **Клиент открывает подписку** `https://sub.example.com/{shortUuid}/fedarisha-json`. Subscription-page проксирует в backend, backend рендерит xray-json:
   - Для каждого fedarisha-host вызывает `FedarishaSubscriptionService.buildOutboundForHost`.
   - PAK достаётся из кеша (если нет — `ensureCredentials` сходит за свежим), `prefix` подставляется per-user, `storage.type` всегда переписывается на `"s3"`.
7. Клиентский xray поднимает outbound, открывает session-папку в S3, пишет `c_hello`. Node-овский inbound либо просыпается по webhook'у S3, либо находит сессию на следующем `pollIntervalMs`, выдаёт `s_ack`, и начинается обмен.
8. На любые изменения статуса пользователя:
   - `USER.DISABLED/LIMITED/EXPIRED/DELETED` → `revokeForUser` → node удаляет PAK у провайдера → backend чистит `UserMeta.metadata.fedarisha`.
   - `USER.TRAFFIC_RESET/MODIFIED` (status=ACTIVE) → `ensureForUser` (re-issue только если PAK потерян, иначе fast-path probe).

Подробно: [subscription-flow.md](subscription-flow.md), [node-api.md](node-api.md).

## Уровни изоляции (security model)

| Уровень | Кто enforce'ит | Что защищает |
| --- | --- | --- |
| **L1: PAK / S3 ACL** | S3-провайдер (VK Cloud / Selectel) | Пользователь физически не может читать/писать чужие префиксы |
| **L2: server gate** | xray inbound, `IsUserAllowed(userPrefix)` | Отозванный, но ещё не удалённый PAK не получит `s_ack` (закрывает race window между revoke и lifecycle expiration) |
| **L3: E2E AES-GCM** | xray fedarisha-transport, X25519+HKDF | Даже если кто-то получит read-доступ к чужому префиксу (broken-by-design `static`-провайдер) — payload зашифрован |

`static` ломает L1 (все пользователи одни ключи), но L2/L3 остаются. Поэтому `static` подходит только single-tenant.

## Кто хранит состояние

| Хранилище | Что лежит | Кто пишет | Кто читает |
| --- | --- | --- | --- |
| **Panel Postgres** (`user_meta.metadata.fedarisha`) | PAK кеш, привязанный к (userId, inboundTag) | `FedarishaProvisioningRepository.upsertPak` | `ensureCredentials` (fast-path), `buildOutboundForHost`, `revokeForUser` |
| **VK Cloud / мастер-аккаунт** | namespace PAK-имён | `pak.service.createKey` | `pak.service.deleteKey` |
| **Selectel IAM** | service users + credentials | `selectel-pak.service.ensureServiceUser/createS3Credentials` | `selectel-pak.service.deleteCredentialsByName` |
| **S3-бакет** `.fedarisha-pak-state/<basePrefix>.json` | members policy state (Selectel only) | `selectel-pak.service.upsertPolicyMember` | то же |
| **S3-бакет** `<prefix>/<userId>/sessions/<sessionId>/` | сами сессии xray (зашифрованные чанки) | xray client + xray node | то же |
| **Конфиг панели** (`xray_config`) | `inbounds[].settings.storage` (master-credentials, тип провайдера) | админ через UI | `FedarishaPakService.resolveStorage`, `buildOutboundForHost` |

Master-credentials хранятся в xray-config панели как plain string — для secrets-management положите в Vault и пробросьте через ENV (не реализовано из коробки, апстримная фича Remnawave).

## Кто чем владеет (deploy ownership)

- **Бакет под fedarisha — отдельный, дедицированный.** Selectel перезаписывает bucket policy целиком; VK Cloud PAK не конфликтует, но GC сметает любые файлы старше 1 дня в `/sessions/`-префиксах через lifecycle rule.
- **Master-credentials S3 — у node, не у клиента.** Клиентский xray получает только prefix-scoped PAK. Скомпрометированный клиент не повышает privileges.
- **Webhook-endpoint — у node.** Backend подставляет дефолт `http://<nodeAddress>/webhook`, но саму регистрацию (PutBucketNotification) делает xray на старте при `autoSetup: true`.
- **TLS до S3 / до webhook'а — внешний.** TLS-cert/key для webhook'а можно указать в `settings.webhook.tlsCert/tlsKey` (пути внутри node-контейнера), или поднять Caddy/nginx перед нодой.

## Полная цепочка зависимостей версий

См. [build-from-source.md](build-from-source.md). Пин-файлы (`node/.xray-core-version`, `backend/.frontend-version`) фиксируют upstream-теги; нарушение пина — основание перепересобрать всё ниже по цепочке.
