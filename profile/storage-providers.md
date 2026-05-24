# S3-провайдеры для PAK-токенов

Блок `settings.storage` в fedarisha-инбаунде задаёт **где живёт бакет** и **какими ключами в него ходить**. Поле `type` дополнительно сообщает ноде Remnawave-стека, **как** нарезать доступ для каждого юзера.

## Два разных вопроса

**1. Транспорт.** Сам xray-core-fedarisha понимает только `type: "s3"` (плюс `"local"` для отладки). Ему нужны валидные accessKey/secretKey — откуда они взялись, ему всё равно. Если используете транспорт **без панели** — кладёте `type: "s3"` и выписанные руками ключи; этого хватает.

> Регистр у `type` не важен — значение приводится к нижнему при выборе backend'а (`proxy/fedarisha/outbound.go: buildStorage`). Пустой `type` тоже валиден: если задан `bucket` — fallback на `s3`, если задан `localDir` — на `local`. Иначе backend падает на старте с `unsupported storage type ""`.

**2. Автоматическая выдача PAK.** Когда поверх работает Remnawave, нужно сказать ноде, как именно она должна *получить* ключи для каждого нового пользователя. Для этого xray-core принимает три алиаса на `s3`, которые на ноде маппятся в PAK-провайдеры:

```
type = "vkcloud-pak"  → нода зовёт VK Cloud S3 с ?prefixAccess
type = "selectel-iam" → нода создаёт IAM service-user и переписывает bucket policy
type = "static"       → нода ничего не делает, отдаёт всем те же мастер-ключи
```

xray-core про эту разницу ничего не знает — все три аliasов работают как обычный s3-storage. Различие живёт только в `node/src/modules/fedarisha-pak/`.

> Параметры, общие для всех вариантов (`bucket`, `endpoint`, `region`, `prefix`, `sessionsDir`), описаны в [inbound-config.md](inbound-config.md). Этот документ — про `type` и его особенности.

## Какой выбрать

```
используете xray-core-fedarisha без панели?
    → "s3"   (или "static" — это то же самое; ключи выписываете сами)

используете полный Remnawave-стек на VK Cloud?
    → "vkcloud-pak"  (1 S3-вызов на provision, native prefix-isolation)

используете полный Remnawave-стек на Selectel?
    → "selectel-iam"  (3-4 IAM/S3-вызова на provision, лимит ~150 юзеров на бакет)

используете Remnawave с self-hosted S3 без credentials API (MinIO, Garage)?
    → "static" с pathStyle: true   (изоляции нет, но транспорт работает)
```

| Провайдер | `type` | Изоляция между юзерами | Стоимость операции | Потолок пользователей на бакет |
| --- | --- | --- | --- | --- |
| VK Cloud PAK | `vkcloud-pak` | per-prefix | 1 S3-вызов на provision/revoke | ограничен квотой PAK на мастер-аккаунт |
| Selectel IAM | `selectel-iam` | per-prefix | 3–4 IAM/S3-вызова на provision | ~100–150 (20 KB policy ceiling) |
| Static | `static` | **нет** (всем одна пара ключей) | 0 (provision/revoke — no-op) | неограничен |

Реализация всех трёх — в [`node/src/modules/fedarisha-pak/`](https://github.com/Fedarisha/node/tree/main/src/modules/fedarisha-pak).

## `type: "vkcloud-pak"`

VK Cloud Prefix Access Keys — родная фича VK S3. Нода делает один S3-запрос с параметром `?prefixAccess`, мастер-ключи остаются у ноды, юзеру отдаётся PAK, действующий только в пределах заданного префикса.

```jsonc
"storage": {
  "type": "vkcloud-pak",
  "bucket": "vlt-prod",
  "endpoint": "hb.ru-msk.vkcloud-storage.ru",
  "region": "ru-msk",
  "accessKey": "<master-access-key>",
  "secretKey": "<master-secret-key>"
}
```

### Особенность namespace'а

PAK-имена живут в **нэймспейсе мастер-аккаунта**, а не бакета. Это значит: два fedarisha-инбаунда на одном мастере и разных бакетах коллидируют по `UserAlreadyExists`, если используют просто `userUuid` как имя PAK.

Чтобы обойти это, нода добавляет `sha1(inboundTag)[:8]` к имени каждого PAK — внутри fedarisha коллизий нет. Но **один мастер-ключ — это один логический набор пользователей**, и квота PAK на аккаунт делится между всеми вашими fedarisha-бакетами.

## `type: "selectel-iam"`

Selectel S3 + IAM service users. На каждую пару `(panel-user, inbound)` нода создаёт отдельный IAM service user, выдаёт ему S3-ключи и **переписывает bucket policy целиком**, добавляя per-user statement.

```jsonc
"storage": {
  "type": "selectel-iam",
  "bucket": "vlt-prod",
  "endpoint": "s3.ru-1.storage.selcloud.ru",
  "region": "ru-1",
  "prefix": "fed",                          // basePrefix внутри бакета (опц., по умолчанию "")
  "accessKey": "<master-s3-access-key>",    // мастер-ключи бакета, нужны только для PutBucketPolicy
  "secretKey": "<master-s3-secret-key>",
  "iam": {
    "accountId": "580430",                  // числовой ID аккаунта Selectel
    "projectName": "My First Project",
    "projectId": "94974e8b26c442dab96acf05eefbf590",
    "username": "Lorena",                   // service-user с ролью iam-admin
    "password": "<password>",
    "identityUrl": "https://cloud.api.selcloud.ru/identity/v3/auth/tokens", // опц., дефолт указан
    "apiUrl": "https://api.selectel.ru"      // опц., дефолт указан
  }
}
```

`identityUrl` и `apiUrl` нужны только если Selectel переедет на новые URL'ы или у вас стенд с прокси перед API. По умолчанию подставляются продовые эндпоинты из [`selectel-pak.service.ts`](https://github.com/Fedarisha/node/blob/main/src/modules/fedarisha-pak/selectel-pak.service.ts).

### Что узнали на живом бакете (`vlt-test`, 2026-05-24)

- Путь к объекту всегда **path-style** (`https://<endpoint>/<bucket>/...`); virtual-host → `NoSuchBucket`.
- Bucket policy — один `Allow`-statement на пользователя: `Principal: { AWS: ["<service_user_uuid>"] }` и `Resource: arn:aws:s3:::<bucket>/<prefix>/<userName>/*`. Это **единственная** форма, которую Selectel реально проверяет. `${aws:username}` не раскрывается, ARN-принципалы не матчатся.
- Состояние policy хранится отдельным JSON-объектом `<bucket>/.fedarisha-pak-state/<prefix>.json`, потому что `GetBucketPolicy` отдаёт 403 даже мастер-ключам.
- **Лимит 20 KB на policy** ⇒ ~100–150 активных пользователей на бакет. Для большего объёма — несколько инбаундов с разными бакетами.
- IAM service user создаётся с ролью `s3.user` (project-scoped). Роли `object_storage:user` и `object_storage_user` устарели и роняют создание.
- IAM-токен получается **домен-scoped** (`scope: { domain: { name: <accountId> } }`). Project-scoped токены отбиваются 401 от `/iam/v1/*`.

**Бакет должен быть выделен под fedarisha** — адаптер перезаписывает bucket policy целиком при каждой операции `provision`/`revoke`. Любые внешние statement'ы будут затёрты.

## `type: "static"`

Pass-through. Каждому пользователю отдаётся **один и тот же** keypair из конфига; никаких операций на стороне S3 (`createKey`/`deleteKey` — no-op). Panel всё ещё назначает каждому юзеру уникальный `prefix`, но соблюдает его только клиент — сам бакет ничего не enforce'ит.

```jsonc
"storage": {
  "type": "static",
  "bucket": "fedarisha",
  "endpoint": "s3.example.com",
  "region": "us-east-1",
  "accessKey": "<shared-access-key>",
  "secretKey": "<shared-secret-key>",
  "pathStyle": true                  // опц., только для node-side probe; xray-core всегда ходит path-style
}
```

> **Про `pathStyle`.** Поле читает только NestJS-слой ноды (`static-pak.service.ts` → `s3-probe.helper.ts`), чтобы probe-запрос лёг по тому же URL, что и боевой трафик. Сам xray-core безусловно использует path-style для любого S3-endpoint'а (см. `proxy/fedarisha/storage/s3/s3.go`, `o.UsePathStyle = true`), поэтому транспорту дополнительно ничего настраивать не нужно.

### Когда подходит

- Self-hosted MinIO / Garage / другой S3 без credentials-API.
- Single-tenant деплой или полностью доверенная аудитория.
- Быстрая проверка транспорта до того, как поднимать настоящую изоляцию.

### Когда не подходит

Мультитенантные деплои. Любой пользователь видит ровно те же ключи и может читать/перезаписывать чужие prefix-ы. L1 (S3 ACL) сломан by design; L2 (server gate) и L3 (E2E AES-GCM) ещё работают, но это половина модели угроз (см. [architecture.md#уровни-изоляции](architecture.md#уровни-изоляции)).
