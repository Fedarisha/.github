# Provisioning и subscription flow

Подробности того, как бэкенд решает, какой PAK выдать пользователю, как кеширует его, и что попадает в подписку. Если ещё не читал [architecture.md](architecture.md) — там сначала.

## Один цикл целиком

Сценарий: alice активируется, тянет подписку, потом её банят. Ниже — что происходит в коде на каждом шаге.

### Активация: USER.ENABLED → ensureForUser

Бэкенд использует обычный NestJS `EventEmitter` (не CQRS — несмотря на то, что CQRS-инфраструктура в проекте есть). Хендлеры в `FedarishaProvisioningEvents` слушают:

| Событие | Условие | Что делает |
| --- | --- | --- |
| `USER.ENABLED` | `user.status === ACTIVE` | `ensureForUser(userId)` |
| `USER.TRAFFIC_RESET` | `user.status === ACTIVE` | `ensureForUser(userId)` |
| `USER.MODIFIED` | `user.status === ACTIVE` | `ensureForUser(userId)` |
| `USER.DISABLED / LIMITED / EXPIRED / DELETED` | — | `revokeForUser(userId)` |

`USER.CREATED` явно не подписан — следом всегда летит `USER.ENABLED`, на нём всё и провижится. `USER.REVOKED` тоже не подписан — это апстрим-событие для ротации VLESS/Trojan-секретов, fedarisha re-issue делает через `TRAFFIC_RESET`/`MODIFIED`.

Все хендлеры **асинхронные** — saga, выпустившая событие, не блокируется. Поведение при сбое ноды:

- `provision` падает → warning в логи, кеш не обновляется. На ближайшем `buildOutboundForHost` будет retry.
- `revoke` падает → warning, **но запись из UserMeta всё равно удаляется**. Иначе мёртвый PAK останется в кеше и продолжит попадать в подписки.

### ensureForUser: для всех инбаундов сразу

```
existingTags = listPaks(userId).map(p => p.inboundTag)        ← что уже кешировано
findUserFedarishaInbounds(userId)                             ← на что юзер entitled через squad'ы
для каждого inbound, чей tag НЕ в existingTags:
    node    = findFirstNodeByInbound(inbound)
    creds   = ensureCredentials({ userId, userUuid, inbound })
    // ensureCredentials сам считает expectedPrefix = "<basePrefix>/<userId>/"
    // и пишет результат в кеш через repository.upsertPak
```

Тут есть тонкость: `ensureForUser` **пропускает инбаунд, если в кеше уже есть запись с таким `inboundTag` — независимо от того, какой там префикс**. Проверка «префикс актуален и PAK живой» делается позже в `ensureCredentials` (`buildOutboundForHost` зовёт его на каждый fetch подписки). Так event-driven путь не превращается в шквал S3-вызовов на каждом `USER.MODIFIED`, а валидация переезжает на медленный path подписки, где она дешевле амортизируется.

Цена компромисса: если админ поменял `basePrefix` инбаунду и сразу же стрельнул `USER.ENABLED`, prefix-mismatch обнаружится только на следующем fetch'е подписки клиентом, а не здесь.

Если для `configProfileUuid` нет ни одной enabled-ноды — warning, инбаунд молча пропускается. Когда нода появится, ближайший fetch подписки вызовет `ensureCredentials` через `buildOutboundForHost`, и провижн произойдёт там.

### ensureCredentials: где принимаются решения

Это единственное место, где панель реально ходит на ноду за PAK. Используется и из `ensureForUser` (массовое), и из `buildOutboundForHost` (по требованию во время рендера подписки).

Логика по шагам:

**Шаг 1 — есть ли кеш с правильным префиксом?**

```typescript
const expectedPrefix = `${trimmedBasePrefix}/${userId}/`;   // resolveUserPrefix
const cached = getPak(userId, inbound.inboundTag);

if (cached && cached.prefix === expectedPrefix) {
    // → probe
}
```

**Шаг 2 — probe.** Если кеш есть и префикс совпадает, проверяем что PAK ещё работает:

```typescript
const probe = await probeFedarishaUser(node, userUuid, inboundTag, expectedPrefix,
                                       cached.accessKey, cached.secretKey);

if (probe.isOk && probe.exists) return cached;       // живой → отдаём как есть
if (!probe.isOk)                return cached;       // транспортный сбой → не трогаем кеш
// дошли сюда: probe прошёл, но PAK мёртв (revoked, бакет пересоздан) → re-issue
```

Контракт «transport hiccup → keep cache» важен. Если нода моргнула на 30 секунд, клиент **не должен** получить пустую подписку. Старый PAK почти наверняка ещё работает.

**Шаг 3 — prefix changed.** Если кеш есть, но `expectedPrefix !== cached.prefix` (админ поменял `basePrefix` у инбаунда), сначала **revoke по старому префиксу**, потом provision по новому:

```typescript
if (cached && cached.prefix !== expectedPrefix) {
    await revokeFedarishaUser(node, userUuid, inboundTag, cached.prefix);  // best-effort
}
```

Без этого revoke'а на VK Cloud упадём в `UserAlreadyExists` — namespace PAK-имён у мастер-аккаунта общий, имя то же, а prefix новый, провайдер не даёт overwrite.

**Шаг 4 — provision.** Идём на ноду:

```typescript
const r = await provisionFedarishaUser(node, userUuid, inboundTag, expectedPrefix);
if (!r.isOk) return null;       // нода не смогла — хост выпадет из подписки
```

`null` — это сигнал `buildOutboundForHost` пропустить хост. Клиент получит подписку без этого инбаунда; на следующем refetch попытка повторится.

**Шаг 5 — кеш.** Записываем в UserMeta и возвращаем.

### Кеш: что и почему

`user_meta.metadata.fedarisha` — JSONB-поле в Postgres:

```jsonc
{
  "fedarisha-eu": {
    "accessKey": "AKIA…",
    "secretKey": "…",
    "prefix":    "users/main/123/",
    "configProfileUuid": "…",
    "issuedAt": "2026-05-24T12:00:00.000Z"
  },
  "fedarisha-us": { /* … */ }
}
```

Несколько решений в дизайне:

- **Ключ — `inboundTag`, не `configProfileInboundUuid`.** Пересоздание инбаунда с тем же тегом → кеш переиспользуется. Переименование тега → кеш потерян, но при следующем provision сработает `createWithReclaim` (нода найдёт orphan, удалит, выдаст новый).
- **`configProfileUuid` хранится в payload'е.** Нужен `revokeForUser`'у, чтобы зарезолвить ноду даже если инбаунд уже удалён из конфига.
- **`issuedAt` не используется для TTL.** PAK живут пока их явно не отзовут. Поле — для observability и дебага.

### Рендер подписки: buildOutboundForHost

Вызывается генератором `xray-json` для каждого fedarisha-хоста, который попал в выдачу пользователю.

```
для каждого fedarisha-host:
    rawInbound  = xray-config inbound с этим тегом
    baseStorage = rawInbound.settings.storage
    tuning      = rawInbound.settings.tuning

    node = findFirstNodeByInbound(configProfileInboundUuid)
    creds = ensureCredentials({ userId, userUuid, inbound })
    if (!creds) continue              // нода не смогла → пропускаем хост

    outbound = {
        protocol: 'fedarisha',
        settings: {
            storage: {
                type:        's3',                ← всегда переписывается
                bucket:      baseStorage.bucket,
                endpoint:    baseStorage.endpoint,
                region:      baseStorage.region,
                prefix:      creds.prefix,        ← per-user, не basePrefix
                sessionsDir: baseStorage.sessionsDir ?? null,
                accessKey:   creds.accessKey,     ← PAK, не master
                secretKey:   creds.secretKey,
            },
            tuning: tuning ?? null,
        },
        // streamSettings НЕ выставляется
    }
```

Что исчезает из outbound по сравнению с inbound'ом, и почему:

| Поле inbound'а | Почему нет в outbound |
| --- | --- |
| `clients[]`, `userLevel` | Server-side список allowed user.id и fallback level. Клиенту знать не нужно. |
| `webhook` | Серверная фича (S3 → node нотификации). Клиент только пишет/читает S3. |
| `accessKey/secretKey` мастера | Заменены на per-user PAK. |
| `prefix` (basePrefix) | Заменён на `<basePrefix>/<userId>/`. |
| `iam.*` | Селектел-специфика для управления service-юзерами. Клиенту вредна — это креды мастера. |
| `authType` | Серверное поле: указывает ноде PAK-провайдера. Клиент об этом не знает — он получает уже готовые ключи. |

### Почему `type: 's3'`

В inbound'е поле `type` всегда `"s3"` (или `"local"` для отладки), а провайдера PAK выбирает отдельное поле `authType`. Клиентский xray-core-fedarisha разговаривает только с реальным S3, поэтому в outbound подписки переписывается `type: 's3'`, а `authType` отбрасывается полностью.

Это **abstraction boundary**: смена провайдера на ноде (например, `vkcloud-pak` → `selectel-iam`) меняет только `storage.authType` в inbound — клиентский outbound остаётся идентичным, перевыпуск подписок не нужен.

### Почему нет streamSettings

В `xray-json.generator.service.ts` для всех протоколов рендерится `outbound.streamSettings` (TCP/WS/gRPC + TLS + sockopt). Для fedarisha этот блок пропускается:

```typescript
if (host.protocol !== 'fedarisha') {
    outbound.streamSettings = this.buildStreamSettings(host);
}
```

Транспорт fedarisha не использует сетевой endpoint — данные идут через S3. Любой `streamSettings` (включая `xtls`/`reality`) клиентский xray просто проигнорирует, а в конфиге создаст шум.

## Что отдаёт subscription-page

Бэкенд рендерит fedarisha-only xray-json только если client-type зарегистрирован. В sub-page добавлено:

```typescript
const FEDARISHA_CLIENT_TYPES: readonly string[] = ['fedarisha-json'];

const isKnownClientType =
    REQUEST_TEMPLATE_TYPE_VALUES.includes(clientType) ||
    FEDARISHA_CLIENT_TYPES.includes(clientType);
```

При `GET /{shortUuid}/fedarisha-json`:

1. Контроллер sub-page видит client-type → принимает `fedarisha-json`.
2. Проксирует в backend: `axios.getSubscription(clientIp, shortUuid, headers, true, 'fedarisha-json')`.
3. Backend-генератор рендерит xray-json **только** с fedarisha-outbound'ами; VLESS/Trojan отфильтрованы по протоколу.
4. Sub-page отдаёт JSON как есть, без HTML-обёртки.

Этот URL зашит в форки клиентов (`v2rayN`-fork, `v2rayNG`-fork). Пользователь кладёт его в «Subscription URL», клиент периодически refetch'ит. **Дефолтный интервал у клиентов — 12 часов.** Без event-driven `ensureForUser` юзер до 12 часов тянул бы пустую подписку после активации.

## Webhook defaults

Когда бэкенд собирает xray-config для ноды (через `start-all-nodes-by-profile.processor`), вызывается `applyFedarishaWebhookDefaults`:

```typescript
const PAK_AUTH_TYPES = new Set(['vkcloud-pak', 'selectel-iam', 'static']);

// isS3Storage = storage.type === 's3' && PAK_AUTH_TYPES.has(storage.authType)
if (isFedarishaInbound && isS3Storage(settings) && typeof settings.webhook === 'object') {
    settings.webhook.enabled    ??= true;
    settings.webhook.listen     ??= ':80';
    settings.webhook.publicUrl  ??= `http://${stripScheme(nodeAddress)}/webhook`;
    settings.webhook.autoSetup  ??= true;
}
```

Три состояния `webhook` в конфиге панели:

- Ключ `webhook` **отсутствует** → бэкенд ничего не добавляет, webhook отключён.
- `"webhook": {}` (пустой объект) → бэкенд заполняет дефолты.
- `"webhook": { "enabled": false }` → тоже отключён (явное `false` не перезаписывается).

Дефолты подставляются только если inbound едет через S3-транспорт с настроенным PAK-провайдером — `type: "local"` или отсутствующий `authType` дефолтов не получают (нет S3-источника событий или некому слать нотификации).

## Валидация на стороне панели

Zod-схема `FedarishaProtocolOptionsSchema` (поле discriminator `protocol === 'fedarisha'`):

```typescript
storage: z.object({
  type:      z.string(),               // not enum
  bucket:    z.string(),
  endpoint:  z.string(),
  region:    z.string(),
  prefix:    z.string(),
  sessionsDir: z.string().nullable(),
  accessKey: z.string(),
  secretKey: z.string(),
}),
tuning: FedarishaTuningOptionsSchema.nullable(),
```

Schema **не проверяет**, что `type` входит в whitelist (поле `authType` тоже не валидируется — оно живёт только на ноде). Сделано намеренно — чтобы добавить новый PAK-провайдер без peer-update бэкенда.

Но whitelist всё-таки есть, в двух местах:

- `applyFedarishaWebhookDefaults` не подставит дефолты, если `storage.type !== 's3'` или `storage.authType` не из `{vkcloud-pak, selectel-iam, static}`.
- Нодовый `resolveProviderKind` строгий: возвращает `isOk: false, error: "storage.authType must be explicitly set to one of …"` для любого `authType` вне того же набора.

## Failure modes

Что увидит оператор и где смотреть:

| Симптом | Где смотреть | Что произошло |
| --- | --- | --- |
| Подписка пустая, в backend logs warning `node …unreachable` | backend logs | Нода офлайн, хост выпал из подписки. На следующем refetch (после восстановления) хост вернётся. |
| Подписка пустая, в logs `inbound … not found in xray config` | node logs | Бэкенд дёрнул provision на `inboundTag`, которого нет в `xray_config` ноды. Config-profile не опубликован или нода читает старый. |
| Подписка отдаётся, но клиент не подключается | xray node logs | Чаще всего — не настроен webhook, и инбаунд не реагирует на новые сессии. Проверьте `pollIntervalMs` (≤ 500ms) или включите webhook. |
| Подписка со старыми ключами после revoke | sub-page → backend → UserMeta | Probe вернул `isOk: false` (нода не отвечала) → бэкенд сохранил старый кеш. Дождитесь восстановления и следующего refetch. |
| `UserAlreadyExists` в node logs на VK Cloud | node logs | Два инбаунда с одним мастер-ключом и разными бакетами без discriminator'а. Перепересоберите node с актуальной версией (sha1-suffix добавляется в `buildPakUserName`). |
| Provision fails на Selectel с `PolicyTooLarge` | node logs + Selectel S3 console | Достигнут 20 KB лимит bucket policy (~100–150 юзеров). Поднимите ещё один инбаунд с другим бакетом. |
