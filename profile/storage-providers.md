# S3-провайдеры для PAK-токенов

Блок `settings.storage` в fedarisha-инбаунде выбирает, как node выдаёт пользователю prefix-scoped S3-доступ. Переключение по обязательному полю `type` — провайдер по умолчанию не назначается, неуказанный или неизвестный `type` приводит к ошибке provision/revoke. Реализация — [`node/src/modules/fedarisha-pak/`](https://github.com/Fedarisha/node/tree/main/src/modules/fedarisha-pak).

| Провайдер | `type` | Изоляция между юзерами | Стоимость операции | Потолок пользователей на бакет |
| --- | --- | --- | --- | --- |
| VK Cloud PAK | `vkcloud-pak` | per-prefix | 1 S3-вызов на provision/revoke | ограничен квотой PAK на мастер-аккаунт |
| Selectel IAM | `selectel-iam` | per-prefix | 3–4 IAM/S3-вызова на provision | ~100–150 (20 KB policy ceiling) |
| Static | `static` | **нет** (всем одна пара ключей) | 0 (provision/revoke — no-op) | неограничен |

## `type: "vkcloud-pak"`

VK Cloud Prefix Access Keys. Node создаёт PAK через S3-расширение `?prefixAccess` мастер-ключами.

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

**Особенность:** PAK-имена живут в нэймспейсе мастер-аккаунта, а не бакета — два fedarisha-инбаунда на одном мастере и разных бакетах коллидируют по `UserAlreadyExists`. Node автоматически добавляет `sha1(inboundTag)[:8]` к имени PAK, так что внутри fedarisha коллизий нет, но один мастер-ключ → один логический набор пользователей.

## `type: "selectel-iam"`

Selectel S3 + IAM service users. Node создаёт по одному IAM service user на пару (panel-user, inbound), выдаёт ему S3-ключи и держит bucket policy с per-user statement-ами.

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
    "password": "<password>"
  }
}
```

### Подкапотные ограничения (verified against `vlt-test` 2026-05-24)

- путь к объекту всегда **path-style** (`https://<endpoint>/<bucket>/...`); virtual-host даёт `NoSuchBucket`;
- bucket policy состоит из одного `Allow`-statement на пользователя: `Principal: { AWS: ["<service_user_uuid>"] }` и `Resource: arn:aws:s3:::<bucket>/<prefix>/<userName>/*` — это единственная форма, которую Selectel реально проверяет (`${aws:username}` не раскрывается, ARN-принципалы не матчатся);
- состояние policy хранится отдельным JSON-объектом `<bucket>/.fedarisha-pak-state/<prefix>.json`, потому что `GetBucketPolicy` отдаёт 403 даже мастер-ключам;
- лимит **20 KB на policy** ⇒ ~100–150 активных пользователей на бакет; для большего объёма — несколько inbound-ов с разными бакетами;
- IAM service user создаётся с ролью `s3.user` (project-scoped); `object_storage:user` / `object_storage_user` устарели и роняют создание;
- IAM-токен получается **домен-scoped** (`scope: { domain: { name: <accountId> } }`); project-scoped токены отбиваются 401 от `/iam/v1/*`.

**Бакет должен быть выделен под fedarisha** — адаптер перезаписывает bucket policy целиком при каждой операции `provision`/`revoke`.

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
  "pathStyle": true                  // опц., нужно для MinIO/Garage без DNS-маршрутизации
}
```

Когда подходит:

- self-hosted MinIO/Garage / другой S3, у которого нет credentials-API;
- single-tenant деплой или полностью доверенная аудитория;
- быстрая проверка транспорта до того, как поднимать настоящую изоляцию.

**Когда не подходит:** мультитенантные деплои. Любой пользователь видит ровно те же ключи и может читать/перезаписывать чужие prefix-ы.
