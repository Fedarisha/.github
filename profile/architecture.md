# Архитектура

Здесь два слоя, и они независимы:

1. **Транспорт** — Xray-core-fedarisha. Это форк xray-core с одним новым протоколом `fedarisha`, который ходит через S3-бакет вместо сокета. Самодостаточен.
2. **Management поверх транспорта** — Remnawave-стек (node + backend + subscription-page). Автоматизирует то, что в standalone-режиме делалось бы вручную: выдача per-user PAK-ключей, lifecycle юзеров, рассылка подписок.

Дальше — про каждый слой отдельно.

---

# Слой 1. Xray-core-fedarisha — транспорт сам по себе

Это **обычный xray-core с одним нестандартным протоколом**. Если разбираться, как он устроен — представляйте его не как «часть VPN-панели», а как «как gRPC, только через файлы в бакете».

## Что он даёт

- Сервер (любой xray с инбаундом `protocol: fedarisha`) и клиент (любой xray с outbound'ом того же протокола) **не открывают сокет друг к другу**. Они читают/пишут одни и те же объекты в одной папке S3-бакета.
- Между ними — handshake X25519 + AES-256-GCM, поверх которого крутится yamux-мультиплекс. Снаружи в бакете видны только зашифрованные кадры, имена которых — порядковые номера (`c_00000000`, `s_00000000`, …).
- Полная спецификация wire-протокола, формат кадров, шифрование, polling, webhook — в [protocol.md](protocol.md).

## Что для этого нужно

Минимальный сетап (без какой-либо панели):

1. **S3-бакет**, к которому есть доступ у обеих сторон. Любой S3-compatible: VK Cloud, Selectel, MinIO, Garage, AWS — что угодно. У нашего сервера будут master-ключи; у клиента — либо те же мастер-ключи (`type: "static"` сценарий), либо отдельный prefix-scoped ключ, который вы выдадите ему сами.
2. **xray-core-fedarisha** на сервере с одним fedarisha-инбаундом:
   ```jsonc
   {
     "tag": "fedarisha-eu",
     "protocol": "fedarisha",
     "settings": {
       "storage": {
         "type":      "s3",
         "bucket":    "fedarisha-eu",
         "endpoint":  "hb.ru-msk.vkcloud-storage.ru",
         "region":    "ru-msk",
         "accessKey": "<master S3 access>",
         "secretKey": "<master S3 secret>"
       },
       "clients": []                             // пустой = single-user
     }
   }
   ```
3. **xray-core-fedarisha** на клиенте с outbound'ом и тем же storage-блоком (или со своим prefix-scoped ключом).

Всё. После запуска обе стороны начнут писать/читать `<bucket>/sessions/<sessionId>/c_*` и `s_*`.

Полная схема инбаунда (включая `tuning` и `webhook`) — в [inbound-config.md](inbound-config.md). Описание блока `storage` — в [storage-providers.md](storage-providers.md), раздел про `type: "s3"` и `type: "static"`.

## Что он НЕ даёт

Транспорт сам по себе:

- **Не ротирует ключи.** Если хотите, чтобы каждый пользователь получил свой prefix-scoped key — выдавайте сами и подкладывайте в его outbound-конфиг.
- **Не знает про «пользователей панели».** `settings.clients` на инбаунде — это server-side список разрешённых `user.id`, который вы редактируете руками (или оставляете пустым для single-user).
- **Не рендерит подписок.** Клиент получает свой xray-config файлом — как вы его доставите, дело ваше.
- **Не следит за тем, кому когда нужно отозвать доступ.** Lifecycle юзеров — на вас.

Для одного-двух пользователей всё это делается руками за пять минут. Для сотен — становится больно. Тут и появляется второй слой.

## Уровни изоляции, которые есть уже на этом слое

Даже в standalone-режиме fedarisha даёт два рубежа защиты данных пользователя:

- **L2: server gate.** Инбаунд перед ответом `s_ack` зовёт `IsUserAllowed(userPrefix)`. В standalone-режиме функция всегда возвращает `true` (single-tenant), но интерфейс есть — Remnawave-слой подключает к нему ACL.
- **L3: E2E AES-256-GCM.** Каждая сессия шифруется ключом, выведенным из X25519-handshake'а. Чтение S3-объектов в обход сессии не даёт plaintext.

Третий уровень — **L1 (PAK / S3 ACL)** — это уже задача того, кто выдаёт пользователю S3-ключи. В standalone-режиме его обеспечиваете вы (выдавая каждому юзеру отдельный prefix-scoped key); в Remnawave-режиме — нода через PAK-провайдера.

---

# Слой 2. Remnawave-стек поверх транспорта

Теперь у вас не один-два юзера, а сотня, они появляются и исчезают, и вы хотите, чтобы каждый получил свой scoped S3-ключ и свежую подписку без вашего участия. Это работа Remnawave-стека.

## Что добавляется

| Кусок | Что делает | Репо |
| --- | --- | --- |
| **panel (backend)** | Кеширует выданные PAK в `user_meta.metadata.fedarisha`, слушает USER.*-события и зовёт ноды, рендерит подписки. | [backend](https://github.com/Fedarisha/backend), модуль `src/modules/fedarisha-provisioning/` |
| **node** | Отвечает на `/node/fedarisha/{provision,revoke,probe}-user`, ходит в S3-провайдера (VK Cloud / Selectel / static) за PAK и параллельно добавляет/убирает пользователя в живой xray-runtime. | [node](https://github.com/Fedarisha/node), модуль `src/modules/fedarisha-pak/` |
| **subscription-page** | Регистрирует client-type `fedarisha-json` и проксирует subscription-запрос в backend. | [subscription-page](https://github.com/Fedarisha/subscription-page) |

Каждая из этих частей — **отдельный сервис поверх транспорта**. Если завтра удалить node — xray-core-fedarisha продолжит работать с теми ключами, которые уже выписаны.

## Жизнь одного пользователя в полном стеке

Возьмём вымышленную Алису (`user.id = 123`, `user.uuid = 8a3c2f1e-…`) и проследим, что происходит между моментами «админ создал в панели» и «клиент Алисы пишет первый кадр».

### Шаг 1. Админ создаёт пользователя в панели

В UI панели Алиса привязана к Internal Squad, в котором есть fedarisha-инбаунд `fedarisha-eu`. Inbound сконфигурирован с PAK-провайдером (не `s3`!) — например, `vkcloud-pak`:

```jsonc
{
  "tag": "fedarisha-eu",
  "protocol": "fedarisha",
  "settings": {
    "storage": {
      "type":   "vkcloud-pak",                    // ← node-side тип, говорит «нода, ходи за PAK в VK Cloud»
      "bucket": "vlt-fedarisha-eu",
      "endpoint": "https://hb.ru-msk.vkcloud-storage.ru",
      "region": "ru-msk",
      "prefix": "users/main",                     // basePrefix для per-user префиксов
      "accessKey": "<master S3 access>",          // только для ноды
      "secretKey": "<master S3 secret>"
    },
    "webhook": {}                                 // дефолты подставит backend
  }
}
```

Бэкенд получает `USER.CREATED → USER.ENABLED` через `EventEmitter`. Хендлер `FedarishaProvisioningEvents` вызывает `ensureForUser(123)`.

### Шаг 2. Бэкенд просит ноду выдать PAK

`ensureForUser` находит все fedarisha-инбаунды Алисы (у нас один) и для каждого делает HTTP-вызов к соответствующей ноде:

```http
POST https://node-eu:3001/node/fedarisha/provision-user
Authorization: Bearer <NODE_JWT>
Content-Type: application/json

{
  "userUuid":   "8a3c2f1e-…",
  "inboundTag": "fedarisha-eu",
  "prefix":     "users/main/123/"
}
```

Префикс — это `basePrefix + userId`. Именно к этой папке (и только к ней) у Алисы появится доступ.

### Шаг 3. Нода выдаёт PAK у S3-провайдера

`FedarishaPakService` на ноде смотрит на `storage.type` инбаунда:

- `vkcloud-pak` → один S3-вызов с параметром `?prefixAccess`. VK Cloud возвращает `accessKey/secretKey`, действующие только в пределах указанного префикса.
- `selectel-iam` → создаёт IAM service user, бьёт его в bucket policy, выдаёт S3-credential.
- `static` → отдаёт мастер-ключи без изменений (single-tenant pass-through; провайдер «выбран», но никакого provisioning'а на S3 не делается).

Кроме PAK нода делает вторую вещь: добавляет Алису как `user.id` в живой xray-runtime через xtls gRPC API (`addTrojanUser` — Trojan здесь как пустой carrier, см. [node-api.md](node-api.md#интеграция-с-xray-runtime-ноды)). Это «открывает дверь» — теперь L2 server gate начнёт пропускать её сессии.

Ответ ноды:

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
    "prefix":    "users/main/123/",
    "configProfileUuid": "…",
    "issuedAt": "2026-05-24T12:00:00.000Z"
  }
}
```

Ключ — `inboundTag`. Это значит: пересоздание инбаунда с тем же тегом сохраняет кеш; переименование тега — теряет (но при следующем provision сработает reclaim).

### Шаг 5. Клиент запрашивает подписку

Алиса открывает в форк-клиенте (`v2rayN`/`v2rayNG`) ссылку:

```
https://sub.example.com/<alice shortUuid>/fedarisha-json
```

Этот URL обрабатывает `subscription-page`. Контроллер видит client-type `fedarisha-json`, проксирует запрос в backend, тот рендерит xray-config — но только из fedarisha-хостов, остальные протоколы отфильтрованы.

Для каждого fedarisha-хоста бэкенд зовёт `buildOutboundForHost`, который собирает outbound из двух частей:

- **Базовая часть** — берётся из xray-config инбаунда: bucket, endpoint, region, tuning.
- **Per-user часть** — `prefix` подставляется Алисин, `accessKey/secretKey` берутся из кеша. Если кеша нет или PAK не прошёл probe — `ensureCredentials` сходит на ноду за свежим прямо во время рендера подписки.

Важный момент: в outbound клиента `storage.type` всегда `"s3"`, независимо от того, что стоит у инбаунда. PAK-провайдеры (vkcloud-pak, selectel-iam, static) — это **серверное** понятие; клиентский xray их не знает и не должен знать. Это даёт свободу: смена провайдера на ноде не требует перевыпуска подписок.

### Шаг 6. Клиент пишет в S3

Дальше — обычный transport-сценарий из Слоя 1. Клиентский xray поднимает outbound, генерирует `sessionId`, создаёт в бакете `c_hello`. Инбаунд видит сессию (через webhook или polling), отвечает `s_ack`. Поехали.

Разница со standalone-режимом — только в том, что Алиса пишет в **свою** папку (`users/main/123/sessions/…`) **своим** ключом, а не общим мастером.

### Шаг 7. Что бывает, когда админ что-то меняет

| Событие в панели | Что делает бэкенд |
| --- | --- |
| `USER.DISABLED / LIMITED / EXPIRED / DELETED` | `revokeForUser` — нода удаляет PAK у провайдера, бэкенд чистит кеш |
| `USER.TRAFFIC_RESET / MODIFIED` (если `status=ACTIVE`) | `ensureForUser` — для каждого инбаунда: если PAK ещё жив (probe) — оставляем, иначе перевыпускаем |
| Сменился `basePrefix` инбаунда | На следующем `ensureCredentials`: revoke старого PAK → provision новый. Без revoke на VK Cloud упадём в `UserAlreadyExists` |
| Удалили ноду из squad'а | Хост выпадает из подписки на следующем рендере. Кеш сам по себе не чистится — это намеренно, при возврате ноды кеш переиспользуется |

Подробнее про events и кеш — [subscription-flow.md](subscription-flow.md).

---

# Reference

## Где живёт состояние

Состояние форка разбросано по четырём местам. Когда дебагаешь — ищи здесь:

| Где | Что | Кто пишет | Кто читает |
| --- | --- | --- | --- |
| **Postgres панели** (`user_meta.metadata.fedarisha`) | PAK-кеш `{accessKey, secretKey, prefix, configProfileUuid, issuedAt}` per `(userId, inboundTag)` | `FedarishaProvisioningRepository.upsertPak` | `ensureCredentials` (fast-path), `buildOutboundForHost`, `revokeForUser` |
| **VK Cloud / Selectel мастер-аккаунт** | PAK-юзеры с правами на префиксы | `pak.service.createKey` / `selectel-pak.service.ensureServiceUser` | те же сервисы при delete |
| **S3-бакет**, объект `.fedarisha-pak-state/<basePrefix>.json` | Список IAM-юзеров и их credentials (Selectel only — bucket policy перезаписывается целиком, и это единственный способ не потерять прежних members) | `selectel-pak.service.upsertPolicyMember` | то же |
| **S3-бакет**, `<bucket>/<basePrefix>/<userId>/sessions/<sessionId>/` | Зашифрованные кадры сессий | xray client + xray node | то же |

Нода **локального state не держит** — никаких баз, никаких маппингов. Если бэкенд потеряет кеш — на следующем provision сработает `createWithReclaim` (нода увидит, что PAK уже есть, удалит и пересоздаст). Если бэкенд и нода рассинхронизировались по сессиям — S3 lifecycle через 24 часа всё подметёт.

## Уровни изоляции (полная картина)

В Remnawave-режиме между Алисой и Бобом стоят три барьера:

**L1: PAK / S3 ACL.** Алиса физически не может прочитать `users/main/<bob id>/`. Это enforce'ит S3-провайдер. Появляется только когда нода выдаёт scoped PAK — в standalone-режиме отсутствует (все ходят под мастером или вы выдаёте scoped-ключи вручную).

**L2: server gate в xray-инбаунде.** Перед тем как ответить `s_ack`, инбаунд проверяет `IsUserAllowed(userPrefix)`. Если PAK был отозван секунду назад — Алиса уже не сможет открыть сессию, даже если S3-ключ ещё технически работает. Это закрывает race window между revoke и реальным закрытием доступа. **Есть и в standalone-режиме** (просто там всегда возвращает true).

**L3: E2E AES-256-GCM.** Даже если кто-то получит read-доступ к чужому префиксу (например, в `static`-режиме) — payload зашифрован. Ключ выводится из X25519-handshake'а. **Есть и в standalone-режиме.**

`static` ломает L1 by design. Но L2/L3 на месте — поэтому `static` остаётся пригодным для single-tenant отладочных setup'ов.

## Где какой код

| Слой | Репозиторий | Что добавляет форк | Главные пути |
| --- | --- | --- | --- |
| Транспорт | [`Xray-core-fedarisha`](https://github.com/Fedarisha/Xray-core-fedarisha) | Сам транспорт fedarisha: listener/dialer, S3-store, crypto, webhook-registry | `proxy/fedarisha/` |
| Remnawave | [`node`](https://github.com/Fedarisha/node) | NestJS-модуль `FedarishaPakModule` — REST + три PAK-провайдера | `src/modules/fedarisha-pak/` |
| Remnawave | [`backend`](https://github.com/Fedarisha/backend) | `FedarishaProvisioningModule` (events + кеш), `FedarishaSubscriptionService` (рендер), webhook-defaults, Zod-схема | `src/modules/fedarisha-provisioning/`, `src/common/utils/apply-fedarisha-webhook-defaults.ts` |
| Remnawave | [`subscription-page`](https://github.com/Fedarisha/subscription-page) | Регистрация client-type `fedarisha-json` | `backend/src/modules/root/root.controller.ts` |
| Клиенты | [`v2rayN`](https://github.com/Fedarisha/v2rayN), [`v2rayNG`](https://github.com/Fedarisha/v2rayNG) | Форки клиентов с встроенным xray-core-fedarisha и пониманием `fedarisha-json` | импортируют ядро как submodule |

## Что важно знать про деплой

Это про Remnawave-сценарий (в standalone большая часть оговорок неактуальна).

**Бакет под fedarisha — отдельный, дедицированный.** Не складывайте туда ничего полезного. Selectel перезаписывает bucket policy целиком на каждое изменение members, а xray ставит lifecycle-rule, который удаляет всё в `/sessions/`-префиксах старше 24 часов.

**Master S3 credentials живут только на ноде.** В UI панели они лежат в xray-config как plain text — клиенту не уходят никогда. Клиентский xray получает только prefix-scoped PAK; компрометация клиента не повышает привилегий до мастера.

**Webhook'и поднимает нода.** Бэкенд через `applyFedarishaWebhookDefaults` подставляет в конфиг ноды дефолты (`:80`, `/webhook`, `autoSetup: true`). Сама же регистрация (PutBucketNotification у S3-провайдера) делается xray на старте инбаунда.

**TLS для webhook'а — на усмотрение оператора.** Можно прописать `settings.webhook.tlsCert/tlsKey` (пути внутри node-контейнера) или поставить Caddy/nginx перед нодой. VK Cloud валидирует подпись HMAC-цепочкой, xray-listener отвечает на неё автоматически.

## Дальше

- **[protocol.md](protocol.md)** — что лежит в бакете, как устроен handshake, как фреймится трафик. Чистая спецификация транспорта.
- **[inbound-config.md](inbound-config.md)** — схема fedarisha-инбаунда в xray-config.
- **[storage-providers.md](storage-providers.md)** — как сконфигурить `storage` (общий S3 для standalone, PAK-провайдеры для Remnawave).
- **[node-api.md](node-api.md)** — REST-контракт panel ↔ node.
- **[subscription-flow.md](subscription-flow.md)** — детально про events, ensureCredentials, рендер outbound, failure modes.
- **[build-from-source.md](build-from-source.md)** — собрать форк руками.
