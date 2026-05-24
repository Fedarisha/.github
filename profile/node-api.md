# Node REST API

Три POST-эндпоинта, через которые панель управляет PAK-кредами на нодах. Это вся поверхность интеграции «backend ↔ node» в части fedarisha — больше панель в ноду не лезет.

Имплементация — [`node/src/modules/fedarisha-pak/`](https://github.com/Fedarisha/node/tree/main/src/modules/fedarisha-pak), бэкенд-сторона (контракт + axios-вызовы) — [`backend/src/common/axios/`](https://github.com/Fedarisha/backend/tree/main/src/common/axios).

## Общие правила транспорта

- Все эндпоинты — `POST`, JSON-body, JSON-response.
- Префикс пути: `/node/fedarisha/`.
- Авторизация — `JwtDefaultGuard` (тот же node JWT, что для всех node-API; backend кладёт `Authorization: Bearer <NODE_SECRET>`).
- HTTP-код всегда `200`. **Бизнес-результат — в `response.isOk`.**
- Все вызовы идемпотентны на стороне провайдера (revoke 404 → ok, повторный provision на тот же ключ → reclaim).

## POST /node/fedarisha/provision-user

Выдать PAK для пары `(userUuid, inboundTag)`.

**Request:**
```jsonc
{
  "userUuid":   "8a3c2f1e-...-...",       // UUID пользователя из панели
  "inboundTag": "fedarisha-eu",            // tag fedarisha-инбаунда в xray-config ноды
  "prefix":     "users/main/123/"          // куда писать PAK (basePrefix + userId)
}
```

**Success:**
```jsonc
{
  "response": {
    "isOk":      true,
    "accessKey": "AKIA…",                  // S3 access key для клиента
    "secretKey": "…",                      // S3 secret key
    "error":     null
  }
}
```

**Failure:**
```jsonc
{
  "response": {
    "isOk":      false,
    "accessKey": null,
    "secretKey": null,
    "error":     "fedarisha inbound fedarisha-eu not found in xray config"
  }
}
```

### Что происходит на ноде

1. `FedarishaPakService.resolveStorage` находит `inbound.settings.storage` в xray-config. Если нет такого тега или type невалиден — `isOk: false`.
2. По `storage.type` выбирается провайдер: `vkcloud-pak` → `PakService`, `selectel-iam` → `SelectelPakService`, `static` → `StaticPakService`.
3. `buildPakUserName(userUuid, inboundTag)` собирает имя `${userUuid}-${sha1(inboundTag)[:8]}` — это чтобы один и тот же `userUuid` на разных инбаундах не конфликтовал в shared namespace VK Cloud.
4. Если PAK уже есть на провайдере, но панель его потеряла — отрабатывает **createWithReclaim**: warning, delete старого, retry create.
5. Параллельно нода добавляет пользователя в живой xray-runtime через xtls gRPC API (см. [Интеграция с xray-runtime](#интеграция-с-xray-runtime-ноды)).

**Таймаут со стороны backend'а:** `20s`.

**Если падает:** backend логирует warning, дропает этот host из подписки (`buildOutboundForHost` возвращает `null`), но не валит весь рендер. На следующем запросе подписки попытка повторится.

## POST /node/fedarisha/revoke-user

Отозвать PAK.

**Request:**
```jsonc
{
  "userUuid":   "8a3c2f1e-...-...",
  "inboundTag": "fedarisha-eu",
  "prefix":     "users/main/123/"        // тот же, что был при provision
}
```

**Response:**
```jsonc
{
  "response": {
    "isOk":  true,
    "error": null
  }
}
```

### Особенности

- **Идемпотентен:** повторный revoke и revoke несуществующего PAK возвращают `isOk: true`.
- **На Selectel:** удаляется S3-credential. Если у service user'а больше нет других credentials — удаляется и сам user, и его member-запись в bucket policy. Если есть другие — user остаётся (другие инбаунды могут на нём висеть).
- **На `static`:** no-op, всегда `isOk: true`.

**Таймаут:** `20s`.

**Если падает:** backend логирует warning, **но не блокирует удаление пользователя в панели**. Это намеренно: лучше оставить orphan PAK у провайдера и почистить вручную, чем не дать админу удалить юзера.

## POST /node/fedarisha/probe-user

Проверить, что выданный ранее PAK ещё работает. Это та операция, которой `ensureCredentials` определяет, можно ли использовать кеш.

**Request:**
```jsonc
{
  "userUuid":   "8a3c2f1e-...-...",
  "inboundTag": "fedarisha-eu",
  "prefix":     "users/main/123/",
  "accessKey":  "AKIA…",                // кешированный PAK
  "secretKey":  "…"
}
```

**Всё ок:**
```jsonc
{ "response": { "isOk": true,  "exists": true,  "error": null } }
```

**PAK мёртв (revoked, prefix сменили, бакет пересоздан):**
```jsonc
{ "response": { "isOk": true,  "exists": false, "error": null } }
```

**Транспортный сбой (S3 503, нода не достучалась):**
```jsonc
{ "response": { "isOk": false, "exists": false, "error": "S3 probe returned unexpected status 503" } }
```

### Что делает provider под капотом

1. PUT пустого объекта на `<prefix>/.rw-pak-probe-<timestamp>-<uuid>` (используя переданные accessKey/secretKey).
2. HEAD на этот же объект.
3. DELETE объекта (failures на DELETE логируются как warning, но не валят probe).

PUT/HEAD → `200-299` → `exists: true`. `403/404` → `exists: false`. Любой другой статус → `isOk: false`.

**Таймаут:** `10s` (короче, чем у provision/revoke — probe должен быть быстрым).

### Как backend интерпретирует результат

- **`exists: true, isOk: true`** → fast-path, кеш ок, отдаём текущий PAK в подписку.
- **`exists: false, isOk: true`** → PAK мёртв → `ensureCredentials` перевыпускает.
- **`isOk: false`** (transport hiccup) → **кеш сохраняется**, отдаём в подписку текущий PAK. Идея: лучше попробовать старый, чем заблокировать клиента из-за временной недоступности ноды.

## Что валидируется (DTO)

DTO — Zod-схемы из `@libs/contracts/commands`. Поля validated на уровне node-controller'а:

| Поле | Тип | Валидация |
| --- | --- | --- |
| `userUuid` | string | uuid-format |
| `inboundTag` | string | non-empty |
| `prefix` | string | non-empty |
| `accessKey` / `secretKey` (только probe) | string | non-empty |

Незаполненные/неверные поля → HTTP 400 с Zod-ошибкой (вне формата `response.isOk`).

## Где живёт state на стороне ноды

| Провайдер | State |
| --- | --- |
| `vkcloud-pak` | namespace PAK-имён в мастер-аккаунте VK Cloud. Локального state нода не держит. |
| `selectel-iam` | IAM service users в Selectel + members policy в `<bucket>/.fedarisha-pak-state/<basePrefix>.json`. Плюс bucket policy перезаписывается целиком на каждую mutation. |
| `static` | Нет state. Креды — те же, что в конфиге инбаунда. |

**Нода не хранит локальной БД** с маппингом `(userUuid, inboundTag) → PAK`. Это знание есть только у backend'а (`user_meta.metadata.fedarisha`). Если backend потеряет state — поможет `createWithReclaim` (revoke + recreate) на следующем `provision-user`.

## Авторизация и сетевой layout

- Node REST по умолчанию слушает на `0.0.0.0:NODE_PORT` (из env). Прячьте за firewall'ом или WireGuard'ом — JWT остаётся, но publicly exposed эндпоинт даёт лишний attack surface.
- `NODE_SECRET` хранится в `.env` и панели, и каждой ноды. Один секрет на пару.
- Для multi-node deployment'а каждая нода может иметь свой `NODE_SECRET` — панель привязывает к node-record'у при регистрации (стандартный Remnawave-флоу).

## Интеграция с xray-runtime ноды

Когда backend вызывает `provision-user`, нода не только создаёт PAK на провайдере, но и должна добавить пользователя в живой xray-config. Делается через xtls gRPC API:

```typescript
xtlsApi.handler.addTrojanUser({
  tag: inboundTag,
  username: userId,        // используется как fedarisha user.id
  password: '',            // пустой, fedarisha не использует
  level: 0,
});
```

**Это не опечатка.** Fedarisha переиспользует протобаф `Trojan.Account` как пустой carrier — у него правильный shape для xray-api `addUser`, но xray-runtime игнорирует password для fedarisha-протокола (auth идёт через PAK + X25519, не через password).

`removeUser` — стандартный `xtlsApi.handler.removeUser(tag, username)`.
