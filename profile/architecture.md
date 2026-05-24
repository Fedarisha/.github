# Архитектура форка

## В одну строку

**Fedarisha — это Remnawave, в котором клиент общается с нодой через S3-бакет вместо TCP.** Всё остальное в этом документе — следствие.

## Зачем

В обычном VPN клиент знает адрес ноды и стучится туда напрямую. DPI это видит, провайдер блокирует адрес — VPN мёртв. Fedarisha делает по-другому:

- Клиент получает доступ к **одной папке в S3-бакете публичного облака** (VK Cloud, Selectel — что угодно).
- Нода читает и пишет в ту же папку.
- DPI видит обычные S3 PUT/GET в адрес `hb.ru-msk.vkcloud-storage.ru`. Не VPN.

Цена — латентность 50–250 мс (читаем файл, а не пакет) и стоимость S3-запросов. Зато блокировка ноды по IP больше не работает: чтобы заблокировать форк, провайдеру нужно отрезать целое облако.

## Жизнь одного пользователя

Самый понятный способ разобраться, как форк устроен — пройти один сценарий целиком. Возьмём пользователя `alice` и проследим всё от «админ нажал создать» до «клиент шлёт первый пакет».

### Шаг 1. Админ создаёт alice в панели

В панели alice привязана к Internal Squad, в котором есть fedarisha-инбаунд `fedarisha-eu`. Inbound сконфигурирован так:

```jsonc
{
  "tag": "fedarisha-eu",
  "protocol": "fedarisha",
  "settings": {
    "storage": {
      "type":   "vkcloud-pak",                              // PAK-провайдер
      "bucket": "vlt-fedarisha-eu",
      "endpoint": "https://hb.ru-msk.vkcloud-storage.ru",
      "region": "ru-msk",
      "prefix": "users/main",                               // basePrefix
      "accessKey": "<master S3 access>",                    // мастер, только для ноды
      "secretKey": "<master S3 secret>"
    },
    "webhook": {}                                           // дефолты подставит backend
  }
}
```

Бэкенд получает `USER.CREATED → USER.ENABLED` через `event-emitter`. Хендлер `FedarishaProvisioningEvents` вызывает `ensureForUser(alice.id)`.

### Шаг 2. Бэкенд просит ноду выдать PAK

`ensureForUser` находит все fedarisha-инбаунды alice (у нас один, `fedarisha-eu`) и для каждого делает HTTP-вызов к соответствующей ноде:

```http
POST https://node-eu:3001/node/fedarisha/provision-user
Authorization: Bearer <NODE_JWT>
Content-Type: application/json

{ "userUuid": "<alice uuid>", "inboundTag": "fedarisha-eu", "prefix": "users/main/<alice id>/" }
```

Префикс — это `basePrefix + userId`. Именно к этой папке (и только к ней) у alice появится доступ.

### Шаг 3. Нода выдаёт PAK у VK Cloud

`FedarishaPakService` на ноде смотрит на `storage.type` инбаунда:

- `vkcloud-pak` → один S3-вызов с параметром `?prefixAccess`. VK Cloud возвращает `accessKey/secretKey`, действующие только в пределах указанного префикса.
- `selectel-iam` → создаёт IAM service user, бьёт его в bucket policy, выдаёт S3-credential.
- `static` → отдаёт мастер-ключи без изменений (single-tenant, для отладки).

Кроме PAK нода делает вторую вещь: добавляет alice как `user.id` в живой xray-runtime через xtls gRPC API (`addTrojanUser` — Trojan здесь как пустой carrier, см. [node-api.md](node-api.md#интеграция-с-xray-runtime-ноды)).

Ответ возвращается бэкенду:

```json
{ "response": { "isOk": true, "accessKey": "AKIA…", "secretKey": "…", "error": null } }
```

### Шаг 4. Бэкенд кеширует PAK

В Postgres, в JSONB-поле `user_meta.metadata.fedarisha`:

```json
{
  "fedarisha-eu": {
    "accessKey": "AKIA…",
    "secretKey": "…",
    "prefix":    "users/main/<alice id>/",
    "configProfileUuid": "…",
    "issuedAt": "2026-05-24T12:00:00.000Z"
  }
}
```

Ключ — `inboundTag`. Это значит: пересоздание инбаунда с тем же тегом сохраняет кеш; переименование тега — теряет (но при следующем provision сработает reclaim).

### Шаг 5. Клиент запрашивает подписку

Alice открывает в форк-клиенте (`v2rayN`/`v2rayNG`) ссылку:

```
https://sub.example.com/<alice shortUuid>/fedarisha-json
```

Этот URL обрабатывает `subscription-page`. Контроллер видит client-type `fedarisha-json`, проксирует запрос в backend, тот рендерит xray-config — но только из fedarisha-хостов, остальные протоколы отфильтрованы.

Для каждого fedarisha-хоста бэкенд зовёт `buildOutboundForHost`, который собирает outbound из двух частей:

- **Базовая часть** — берётся из xray-config инбаунда: bucket, endpoint, region, tuning.
- **Per-user часть** — `prefix` подставляется alice'ин, `accessKey/secretKey` берутся из кеша. Если кеша нет или PAK не прошёл probe — `ensureCredentials` сходит на ноду за свежим прямо во время рендера подписки.

Важный момент: в outbound клиента `storage.type` всегда `"s3"`, независимо от того, что стоит у инбаунда. PAK-провайдеры (vkcloud-pak, selectel-iam, static) — это **серверное** понятие; клиентский xray их не знает и не должен знать. Это даёт свободу: смена провайдера на ноде не требует перевыпуска подписок.

### Шаг 6. Клиент пишет в S3

Клиентский xray поднимает outbound, генерирует `sessionId` (16 случайных байт) и создаёт в бакете объект:

```
vlt-fedarisha-eu/users/main/<alice id>/sessions/<sessionId>/c_hello
```

В теле — публичный X25519-ключ клиента. С точки зрения S3 это обычный PUT, ничем не отличающийся от загрузки фотки.

### Шаг 7. Нода видит сессию

Здесь есть два пути:

- **С webhook'ом** (default). VK Cloud / Selectel настроены слать `s3:ObjectCreated:Put` на endpoint ноды. Нода получает нотификацию, видит новый `c_hello`, отвечает `s_ack`. Латентность хендшейка — меньше 100 мс.
- **Без webhook'а**. Нода поллит `LIST` сессионных папок с интервалом `pollIntervalMs` (по умолчанию 500 мс если webhook есть, 100 мс если нет). Латентность хендшейка — 50–500 мс.

Дальше обмен идёт через нумерованные файлы `c_00000000`, `s_00000000`, … — каждый зашифрован AES-256-GCM сессионным ключом, выведенным из X25519. Подробности в [protocol.md](protocol.md).

### Шаг 8. Что бывает, когда админ что-то меняет

| Событие в панели | Что делает бэкенд |
| --- | --- |
| `USER.DISABLED / LIMITED / EXPIRED / DELETED` | `revokeForUser` — нода удаляет PAK у провайдера, бэкенд чистит кеш |
| `USER.TRAFFIC_RESET / MODIFIED` (если `status=ACTIVE`) | `ensureForUser` — для каждого инбаунда: если PAK ещё жив (probe) — оставляем, иначе перевыпускаем |
| Сменился `basePrefix` инбаунда | На следующем `ensureCredentials`: revoke старого PAK → provision новый. Без revoke на VK Cloud упадём в `UserAlreadyExists` |
| Удалили нода из squad'а | Хост выпадает из подписки на следующем рендере. Кеш сам по себе не чистится — это намеренно, при возврате ноды кеш переиспользуется |

Подробнее про events и кеш — [subscription-flow.md](subscription-flow.md).

---

## Кто что хранит

Состояние форка разбросано по четырём местам. Когда дебагаешь — ищи здесь:

| Где | Что | Кто пишет | Кто читает |
| --- | --- | --- | --- |
| **Postgres панели** (`user_meta.metadata.fedarisha`) | PAK-кеш `{accessKey, secretKey, prefix, configProfileUuid, issuedAt}` per `(userId, inboundTag)` | `FedarishaProvisioningRepository.upsertPak` | `ensureCredentials` (fast-path), `buildOutboundForHost`, `revokeForUser` |
| **VK Cloud / Selectel мастер-аккаунт** | PAK-юзеры с правами на префиксы | `pak.service.createKey` / `selectel-pak.service.ensureServiceUser` | те же сервисы при delete |
| **S3-бакет**, объект `.fedarisha-pak-state/<basePrefix>.json` | Список IAM-юзеров и их credentials (Selectel only — bucket policy перезаписывается целиком, и это единственный способ не потерять прежних members) | `selectel-pak.service.upsertPolicyMember` | то же |
| **S3-бакет**, `<bucket>/<basePrefix>/<userId>/sessions/<sessionId>/` | Зашифрованные кадры сессий | xray client + xray node | то же |

Нода **локального state не держит** — никаких баз, никаких маппингов. Если бэкенд потеряет кеш — на следующем provision сработает `createWithReclaim` (нода увидит, что PAK уже есть, удалит и пересоздаст). Если бэкенд и нода рассинхронизировались по сессиям — S3 lifecycle через 24 часа всё подметёт.

## Уровни изоляции

Между alice и bob'ом стоят три барьера. Если ломается один — два других ещё держат:

**L1: PAK / S3 ACL.** Alice физически не может прочитать `users/main/<bob id>/`. Это enforce'ит S3-провайдер. Самый сильный уровень — без сговора с провайдером не обойти.

**L2: server gate в xray-инбаунде.** Перед тем как ответить `s_ack`, инбаунд проверяет `IsUserAllowed(userPrefix)`. Если PAK был отозван секунду назад — alice уже не сможет открыть сессию, даже если S3-ключ ещё технически работает (TTL у revoke не мгновенный). Это закрывает race window между revoke и реальным закрытием доступа.

**L3: E2E AES-256-GCM.** Даже если кто-то получит read-доступ к чужому префиксу (например, в `static`-режиме все ходят под мастер-ключом) — payload зашифрован. Ключ выводится из X25519-handshake'а, в S3 хранится только publickey.

`static` ломает L1 by design. Но L2/L3 на месте — поэтому `static` остаётся пригодным для single-tenant отладочных setup'ов.

## Где какой код

| Репозиторий | Что добавляет форк | Главные пути |
| --- | --- | --- |
| [`Xray-core-fedarisha`](https://github.com/Fedarisha/Xray-core-fedarisha) | Сам транспорт fedarisha: listener/dialer, S3-store, crypto, webhook-registry | `proxy/fedarisha/` |
| [`node`](https://github.com/Fedarisha/node) | NestJS-модуль `FedarishaPakModule` — REST + три PAK-провайдера | `src/modules/fedarisha-pak/` |
| [`backend`](https://github.com/Fedarisha/backend) | `FedarishaProvisioningModule` (events + кеш), `FedarishaSubscriptionService` (рендер), webhook-defaults, Zod-схема | `src/modules/fedarisha-provisioning/`, `src/common/utils/apply-fedarisha-webhook-defaults.ts` |
| [`subscription-page`](https://github.com/Fedarisha/subscription-page) | Регистрация client-type `fedarisha-json` | `backend/src/modules/root/root.controller.ts` |
| [`v2rayN`](https://github.com/Fedarisha/v2rayN), [`v2rayNG`](https://github.com/Fedarisha/v2rayNG) | Форки клиентов с встроенным xray-core-fedarisha и пониманием `fedarisha-json` | импортируют ядро как submodule |

## Что важно знать про деплой

**Бакет под fedarisha — отдельный, дедицированный.** Не складывайте туда ничего полезного. Selectel перезаписывает bucket policy целиком на каждое изменение members, а xray ставит lifecycle-rule, который удаляет всё в `/sessions/`-префиксах старше 24 часов.

**Master S3 credentials живут только на ноде.** В UI панели они лежат в xray-config как plain text — клиенту не уходят никогда. Клиентский xray получает только prefix-scoped PAK; компрометация клиента не повышает привилегий до мастера.

**Webhook'и поднимает нода.** Бэкенд через `applyFedarishaWebhookDefaults` подставляет в конфиг ноды дефолты (`:80`, `/webhook`, `autoSetup: true`). Сама же регистрация (PutBucketNotification у S3-провайдера) делается xray на старте инбаунда.

**TLS для webhook'а — на усмотрение оператора.** Можно прописать `settings.webhook.tlsCert/tlsKey` (пути внутри node-контейнера) или поставить Caddy/nginx перед нодой. VK Cloud валидирует подпись HMAC-цепочкой, xray-listener отвечает на неё автоматически.

## Дальше

- **[protocol.md](protocol.md)** — что лежит в бакете, как устроен handshake, как фреймится трафик.
- **[node-api.md](node-api.md)** — три REST-эндпоинта ноды, формат запросов/ответов, как ведёт себя при сбоях.
- **[subscription-flow.md](subscription-flow.md)** — детально про events, ensureCredentials, рендер outbound, failure modes.
- **[build-from-source.md](build-from-source.md)** — собрать форк руками.
