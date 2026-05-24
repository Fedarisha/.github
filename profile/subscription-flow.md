# Subscription flow

Как пользователь панели получает рабочую fedarisha-подписку: от событий lifecycle до JSON, который грузит клиент.

## TL;DR

```
USER.ENABLED / TRAFFIC_RESET / MODIFIED   →  ensureForUser(userId)
                                              ├─ для каждого fedarisha-inbound:
                                              │   POST /node/fedarisha/provision-user
                                              └─ сохранить PAK в user_meta.metadata.fedarisha

USER.DELETED / DISABLED / LIMITED / EXPIRED → revokeForUser(userId)
                                              ├─ для каждого кешированного PAK:
                                              │   POST /node/fedarisha/revoke-user
                                              └─ очистить user_meta.metadata.fedarisha

GET https://sub.example.com/{shortUuid}/fedarisha-json
   → subscription-page → backend → xray-json.generator
     → для каждого fedarisha-host: buildOutboundForHost
        ├─ fast-path: probe кеша + переотдать тот же PAK
        └─ miss: ensureCredentials → node provision → store → render
```

## Event-driven provisioning

[`FedarishaProvisioningEvents`](https://github.com/Fedarisha/backend/blob/main/src/modules/fedarisha-provisioning/fedarisha-provisioning.events.ts) подписан на user lifecycle через `@OnEvent` (event-emitter, не CQRS — несмотря на наличие CQRS-инфраструктуры в проекте).

| Событие | Условие | Действие |
| --- | --- | --- |
| `USER.CREATED` | — | (не подписан; реальное провижн идёт через `ENABLED`, которое следом эмитится) |
| `USER.ENABLED` | `user.status === ACTIVE` | `ensureForUser` |
| `USER.TRAFFIC_RESET` | `user.status === ACTIVE` | `ensureForUser` |
| `USER.MODIFIED` | `user.status === ACTIVE` | `ensureForUser` |
| `USER.DISABLED` | — | `revokeForUser` |
| `USER.LIMITED` | — | `revokeForUser` |
| `USER.EXPIRED` | — | `revokeForUser` |
| `USER.DELETED` | — | `revokeForUser` |
| `USER.REVOKED` | — | (используется upstream для ротации credentials VLESS/Trojan; fedarisha re-issue делает через TRAFFIC_RESET / MODIFIED, явный handler не нужен) |

Все handler'ы выполняются **асинхронно** (event-emitter не блокирует saga, выпустившую событие). Если node недоступна:
- `provision` failures → warning в логи, кеш не обновляется, на следующем `buildOutboundForHost` retry;
- `revoke` failures → warning, **запись из UserMeta всё равно удаляется**. Это критично: иначе старый PAK останется в кеше и попадёт в подписку при последующих рендерах.

## ensureForUser (bulk provisioning)

```
1. listPaks(userId)                                   // что уже в кеше
2. findUserFedarishaInbounds(userId)                  // на что юзер entitled
3. для каждого inbound без свежего кеша:
   a. findFirstNodeByInbound(inbound)                 // на какой ноде сейчас
   b. resolveUserPrefix(inbound.basePrefix, userId)   // куда писать
   c. ensureCredentials({ userId, userUuid, inbound, prefix })
   d. upsertPak(userId, inboundTag, response)
```

**Quick-path:** если в кеше уже есть PAK с тем же `prefix` И `probeUser` вернул `exists: true` — заново ничего не дёргаем.

**Prefix mismatch:** если в кеше есть PAK, но `prefix` другой (админ поменял `basePrefix` инбаунда) — **сначала revoke старого**, потом provision нового. Без этого revoke на VK Cloud упадём в `UserAlreadyExists`: namespace PAK-имён в VK Cloud — per-master-account, имя одно и то же, а prefix новый, провайдер не даёт overwrite.

**Без ноды:** если для `configProfileUuid` нет ни одной enabled ноды — warning, inbound пропускается. Когда нода появится, следующий subscription-fetch повторит попытку (ensureCredentials дёрнется внутри `buildOutboundForHost`).

## ensureCredentials (single inbound)

Используется и из `ensureForUser`, и из subscription-render (`buildOutboundForHost`). Это единственная точка, где panel реально ходит в node за PAK.

```typescript
ensureCredentials({ userId, userUuid, inbound }) {
  const expectedPrefix = `${inbound.basePrefix}/${userId}/`;
  const cached = getPak(userId, inbound.inboundTag);

  // 1. fast-path: kosher cache
  if (cached && cached.prefix === expectedPrefix) {
    const probe = await probeFedarishaUser(node, userUuid, inboundTag, expectedPrefix, cached.accessKey, cached.secretKey);
    if (probe.isOk && probe.exists) return cached;     // кеш живой
    if (!probe.isOk) return cached;                    // транспортный hiccup — отдаём что есть
    // else: probe ok, но PAK мёртв (revoked, бакет пересоздан) → re-issue
  }

  // 2. prefix changed → revoke первого, затем re-issue
  if (cached && cached.prefix !== expectedPrefix) {
    await revokeFedarishaUser(node, userUuid, inboundTag, cached.prefix);   // best-effort
  }

  // 3. provision свежий
  const response = await provisionFedarishaUser(node, userUuid, inboundTag, expectedPrefix);
  if (!response.isOk) return null;                     // нода не смогла — host выпадет из подписки

  // 4. cache
  await upsertPak(userId, inboundTag, {
    accessKey: response.accessKey,
    secretKey: response.secretKey,
    prefix: expectedPrefix,
    configProfileUuid: inbound.configProfileUuid,
    issuedAt: new Date().toISOString(),
  });

  return { accessKey, secretKey, prefix: expectedPrefix };
}
```

**Контракт «transport hiccup → keep cache»** важен: если node моргнула на 30 секунд во время probe, клиент **не должен** получить пустую подписку. Старый PAK почти наверняка ещё работает.

## Хранилище кеша

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

Ключ верхнего уровня — `inboundTag`, не `configProfileInboundUuid`. Это даёт два свойства:
- Перерегистрация инбаунда (delete + recreate с тем же тегом) — кеш переиспользуется;
- Переименование тега в xray-config — кеш потерян, на следующем provision сработает createWithReclaim.

`configProfileUuid` лежит в payload'е, чтобы `revokeForUser` мог зарезолвить нужную ноду даже если inbound уже выпилен из конфига.

`issuedAt` не используется для TTL — PAK живут пока их явно не отзовут. Поле — для observability/debug.

## Subscription render

[`FedarishaSubscriptionService.buildOutboundForHost`](https://github.com/Fedarisha/backend/blob/main/src/modules/fedarisha-provisioning/fedarisha-subscription.service.ts) вызывается генератором `xray-json` (и `singbox`-генератором, если поддерживается клиентом) для каждого fedarisha-host'а в подписке.

```
для каждого fedarisha-host:
  rawInbound      = xray-config inbound с этим тегом
  basePrefix      = rawInbound.settings.storage.prefix
  baseStorage     = rawInbound.settings.storage
  tuning          = rawInbound.settings.tuning

  node            = findFirstNodeByInbound(configProfileInboundUuid)
  creds           = ensureCredentials({ userId, userUuid, inbound: { inboundTag, basePrefix, configProfileUuid, nodeAddress, nodePort }})
  if !creds: skip this host

  outbound = {
    protocol: 'fedarisha',
    settings: {
      storage: {
        type:      's3',                          ← всегда переписывается
        bucket:    baseStorage.bucket,
        endpoint:  baseStorage.endpoint,
        region:    baseStorage.region,
        prefix:    creds.prefix,                  ← per-user, не basePrefix
        sessionsDir: baseStorage.sessionsDir ?? null,
        accessKey: creds.accessKey,               ← PAK, не master
        secretKey: creds.secretKey,               ← PAK, не master
      },
      tuning: tuning ?? null,
    },
    // streamSettings НЕ выставляется (см. ниже)
  }
```

### Что выкинуто из outbound

| Поле inbound'а | Почему не в outbound |
| --- | --- |
| `clients[]` | На клиенте нет auth — это server-side список разрешённых user.id |
| `userLevel` | Аналогично — server-side fallback level |
| `webhook` | Серверная фича (S3 → node нотификации). Клиент только пишет/читает S3. |
| `accessKey/secretKey` мастер-ключей | Заменены на per-user PAK |
| `prefix` (basePrefix) | Заменён на per-user `<basePrefix>/<userId>/` |
| `iam.*` | Селектел-специфика, нужна только node для управления credentials |

### Почему `type: 's3'`

`vkcloud-pak`/`selectel-iam`/`static` — это **node-side понятия** (выбор PAK-провайдера). Клиентский xray-core-fedarisha их не знает: его `infra/conf/fedarisha.go` принимает только `s3`/`local` (+ алиасы коллапсятся в s3, но это для совместимости конфигов на сервере, а не для клиента).

Хардкод `type: 's3'` в [`fedarisha-subscription.service.ts`](https://github.com/Fedarisha/backend/blob/main/src/modules/fedarisha-provisioning/fedarisha-subscription.service.ts) — намеренная abstraction boundary: смена провайдера на ноде (vkcloud-pak → selectel-iam) не требует перевыпуска подписок.

### Почему нет `streamSettings`

В [`xray-json.generator.service.ts`](https://github.com/Fedarisha/backend/blob/main/src/modules/subscription-template/generators/xray-json.generator.service.ts) для всех протоколов рендерится `outbound.streamSettings` (TCP/WS/gRPC + TLS + sockopt). **Для fedarisha этот блок пропускается**:

```typescript
// Fedarisha is S3-backed: no transport, no TLS, no streamSettings.
if (host.protocol !== 'fedarisha') {
    outbound.streamSettings = this.buildStreamSettings(host);
}
```

Транспорт fedarisha не использует сетевой endpoint — данные идут через S3. Любой `streamSettings` (включая `xtls`/`reality`) просто проигнорируется клиентским xray и засоряет конфиг.

## Subscription-page

Сам по себе backend отдаёт `xray-json` только если client-type зарегистрирован.

[`subscription-page/backend/src/modules/root/root.controller.ts`](https://github.com/Fedarisha/subscription-page/blob/main/backend/src/modules/root/root.controller.ts) добавляет:

```typescript
const FEDARISHA_CLIENT_TYPES: readonly string[] = ['fedarisha-json'];

// в обработчике @Get([':shortUuid', ':shortUuid/:clientType'])
const isKnownClientType =
    REQUEST_TEMPLATE_TYPE_VALUES.includes(clientType) ||
    FEDARISHA_CLIENT_TYPES.includes(clientType);
```

При `GET /{shortUuid}/fedarisha-json`:

1. Контроллер sub-page проверяет client-type → принимает `fedarisha-json`.
2. Ходит в backend (`axios.getSubscription(clientIp, shortUuid, headers, true, 'fedarisha-json')`).
3. Backend генератор для этого client-type рендерит xray-json **только** с fedarisha-outbound'ами; VLESS/Trojan-хосты отфильтрованы по протоколу.
4. Sub-page возвращает JSON как есть (без HTML-обёртки).

Этот URL зашивается в форки клиентов (v2rayN-fork, v2rayNG-fork) — пользователь кладёт его в "Subscription URL", клиент периодически refetch'ит.

**Refresh interval** клиенты обычно ставят 12h. Поэтому event-driven `ensureForUser` (на `USER.ENABLED`) важен: без него юзер до 12 часов будет тянуть пустую подписку, пока не дёрнет subscription руками.

## Webhook defaults

Когда backend генерирует xray-config для ноды (start-all-nodes-by-profile.processor), вызывается [`applyFedarishaWebhookDefaults`](https://github.com/Fedarisha/backend/blob/main/src/common/utils/apply-fedarisha-webhook-defaults.ts).

```typescript
const S3_STORAGE_TYPES = new Set(['vkcloud-pak', 'selectel-iam', 'static']);

if (isFedarishaInbound && isS3Storage(settings) && settings.webhook is object) {
    settings.webhook.enabled    ??= true;
    settings.webhook.listen     ??= ':80';
    settings.webhook.publicUrl  ??= `http://${stripScheme(nodeAddress)}/webhook`;
    settings.webhook.autoSetup  ??= true;
}
```

Хитрость: если в xray-config панели стоит `"webhook": {}` (пустой объект) — backend заполнит дефолты. Если ключа `webhook` нет вообще — backend ничего не добавляет (webhook отключён). Если `"webhook": { "enabled": false }` — тоже отключён.

S3-провайдеры (vkcloud-pak/selectel-iam/static) — единственные, у кого webhook'и имеют смысл. `localDir`-инбаунды webhook'и не получают (нет S3-источника событий).

## Валидация на стороне панели

[`FedarishaProtocolOptionsSchema`](https://github.com/Fedarisha/backend/blob/main/libs/contract/models/resolved-proxy-config.schema.ts) (Zod, поле discriminator `protocol === 'fedarisha'`):

```typescript
const FedarishaProtocolOptionsSchema = z.object({
  storage: z.object({
    type:      z.string(),               // not enum — алиасы валидируются на node-стороне
    bucket:    z.string(),
    endpoint:  z.string(),
    region:    z.string(),
    prefix:    z.string(),
    sessionsDir: z.string().nullable(),
    accessKey: z.string(),
    secretKey: z.string(),
  }),
  tuning: FedarishaTuningOptionsSchema.nullable(),
});
```

Schema не проверяет, что `type` входит в whitelist — это сделано чтобы добавить новый provider в node без peer-update backend'а. Но `applyFedarishaWebhookDefaults` использует whitelist (`S3_STORAGE_TYPES`), и node `resolveProviderKind` тоже строгий. Неизвестный `type`:
- backend не подставит webhook-дефолты;
- node вернёт `isOk: false, error: "storage.type must be explicitly set to one of …"`.

## Failure modes (что увидит оператор)

| Симптом | Где смотреть | Что произошло |
| --- | --- | --- |
| Подписка пустая, в backend logs warning `node …unreachable` | backend logs | Нода офлайн, host выпал из подписки. Клиент получит fedarisha-outbound на следующем refetch после восстановления. |
| Подписка пустая, в logs `inbound … not found in xray config` | node logs | Backend дёрнул provision на inboundTag, которого нет в `xray_config`. Проверьте опубликован ли config-profile. |
| Подписка отдаётся, но клиент не подключается | xray node logs | Чаще всего — webhook не настроен, и инбаунд не реагирует на новые сессии. Проверьте `pollIntervalMs` (≤ 500ms) или включите webhook. |
| Подписка отдаётся со старыми ключами после revoke | sub-page → backend → UserMeta | Probe вернул `isOk: false` (нода не отвечала) → backend сохранил кеш. Дождитесь восстановления + следующего refetch. |
| `UserAlreadyExists` в node logs на VK Cloud | node logs | Два инбаунда с одним мастер-ключом и разными бакетами без discriminator'а. Перепересоберите node с актуальной версией (sha1-suffix добавлен в pak.service). |
| Provision fails на Selectel с `PolicyTooLarge` | node logs + Selectel S3 console | Достигнут 20 KB лимит bucket policy (~100-150 юзеров). Поднимите ещё один inbound с другим бакетом. |
