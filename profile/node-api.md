# Node REST API

REST-эндпоинты, через которые панель управляет PAK-кредами на нодах. Реализация — [`node/src/modules/fedarisha-pak/`](https://github.com/Fedarisha/node/tree/main/src/modules/fedarisha-pak), backend-сторона — [`backend/src/common/axios/axios.service.ts`](https://github.com/Fedarisha/backend/blob/main/src/common/axios/axios.service.ts).

## Транспорт

- Все эндпоинты — `POST`, JSON-body, JSON-response.
- Префикс пути: `/node/fedarisha/`.
- Авторизация — `JwtDefaultGuard` (тот же node JWT, что для всех node-API; backend кладёт `Authorization: Bearer <NODE_SECRET>` — см. `axios.service.ts`).
- HTTP-код всегда `200`. Бизнес-результат — в `response.isOk`.
- Все вызовы идемпотентны на стороне провайдера (revoke 404 → ok, повторный provision на тот же ключ → reclaim).

## Контракт

### `POST /node/fedarisha/provision-user`

Выдать PAK для пары `(userUuid, inboundTag)`.

**Request:**
```jsonc
{
  "userUuid":   "8a3c2f1e-...-...",     // UUID пользователя из панели
  "inboundTag": "fedarisha-eu",          // tag fedarisha-инбаунда в xray-config
  "prefix":     "users/main/123/"        // куда писать PAK (basePrefix + userId)
}
```

**Response (success):**
```jsonc
{
  "response": {
    "isOk":      true,
    "accessKey": "AKIA…",                // S3 access key для клиента
    "secretKey": "…",                    // S3 secret key
    "error":     null
  }
}
```

**Response (failure):**
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

**Особенности:**
- Если PAK для `(userUuid, inboundTag)` уже существует на провайдере, но панель его потеряла — отрабатывает **createWithReclaim**: ноду логирует warning, удаляет старый PAK и создаёт новый. Возвращает свежие ключи.
- VK Cloud namespace shared между всеми бакетами одного мастер-аккаунта. Чтобы избежать `UserAlreadyExists` между разными инбаундами — node автоматически добавляет `sha1(inboundTag)[:8]` к имени PAK-юзера, **client об этом ничего не знает** (получает плоский accessKey/secretKey).
- На Selectel: создаётся IAM service user, выдаются S3-credentials, апдейтится bucket policy (см. [storage-providers.md#type-selectel-iam](storage-providers.md)).
- На `static`: pass-through мастер-ключей конфига, никаких S3-вызовов.

**Таймаут:** `20s` (со стороны backend'а в `axios.service.ts:781`).

**Backend behavior на failure:** логирует warning, дропает этот host из подписки (`buildOutboundForHost` возвращает `null`), но не валит весь рендер. На следующем запросе подписки попытка повторится.

---

### `POST /node/fedarisha/revoke-user`

Отозвать PAK.

**Request:**
```jsonc
{
  "userUuid":   "8a3c2f1e-...-...",
  "inboundTag": "fedarisha-eu",
  "prefix":     "users/main/123/"        // тот же, что был при provision (на случай миграции)
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

**Особенности:**
- Идемпотентен: повторный revoke и revoke несуществующего PAK возвращают `isOk: true`.
- На Selectel: удаляется S3-credential. Если у service user'а больше нет других credentials — удаляется и сам user, и его member-запись в bucket policy. Если есть другие — user остаётся (другие инбаунды могут на нём висеть).
- На `static`: no-op, всегда `isOk: true`.

**Таймаут:** `20s` (`axios.service.ts:813`).

**Backend behavior на failure:** логирует warning, **не** блокирует удаление пользователя в панели. Это намеренно: лучше оставить orphan PAK у провайдера и почистить вручную, чем не дать админу удалить юзера.

---

### `POST /node/fedarisha/probe-user`

Проверить, что выданный ранее PAK ещё работает.

**Request:**
```jsonc
{
  "userUuid":   "8a3c2f1e-...-...",
  "inboundTag": "fedarisha-eu",
  "prefix":     "users/main/123/",
  "accessKey":  "AKIA…",                 // кешированный PAK
  "secretKey":  "…"
}
```

**Response (всё ок):**
```jsonc
{
  "response": {
    "isOk":   true,
    "exists": true,
    "error":  null
  }
}
```

**Response (PAK невалиден):**
```jsonc
{
  "response": {
    "isOk":   true,
    "exists": false,
    "error":  null
  }
}
```

**Response (transport/infra error):**
```jsonc
{
  "response": {
    "isOk":   false,
    "exists": false,
    "error":  "S3 probe returned unexpected status 503"
  }
}
```

**Что делает provider на пробе:**

1. PUT пустого объекта на `<prefix>/.rw-pak-probe-<timestamp>-<uuid>` (используя переданные accessKey/secretKey).
2. HEAD на этот же объект.
3. DELETE объекта (failures на DELETE логируются как warning, но не валят probe).

Если PUT/HEAD → `200-299` → `exists: true`. Если `403/404` → `exists: false`. Любой другой статус → `isOk: false`.

**Таймаут:** `10s` (`axios.service.ts:842`).

**Backend behavior:**
- `exists: true, isOk: true` → fast-path, кеш ок, отдаём текущий PAK в подписку.
- `exists: false, isOk: true` → PAK мёртв (revoked, prefix сменили, бакет пересоздан) → `ensureCredentials` перевыпускает.
- `isOk: false` (transport hiccup) → **кеш сохраняется**, отдаём в подписку текущий PAK. Идея: лучше попробовать старый, чем заблокировать клиента из-за временной недоступности ноды.

---

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
| `vkcloud-pak` | namespace PAK-имён в мастер-аккаунте VK Cloud. Нода не держит локального state. |
| `selectel-iam` | IAM service users в Selectel + members policy в `<bucket>/.fedarisha-pak-state/<basePrefix>.json`. Plus bucket policy перезаписывается целиком на каждую mutation. |
| `static` | Нет state. Креды — те же, что в конфиге инбаунда. |

В частности, **node не хранит локальной БД** с маппингом `(userUuid, inboundTag) → PAK`. Это знание есть только у backend'а (`user_meta.metadata.fedarisha`). Если backend потеряет state — поможет `createWithReclaim` (revoke + recreate) на следующем `provision-user`.

## Авторизация и сетевой layout

- Node REST по умолчанию слушает на `0.0.0.0:NODE_PORT` (из env). Прячьте за firewall'ом или WireGuard'ом — JWT остаётся, но publicly exposed эндпоинт даёт лишний attack surface.
- `NODE_SECRET` хранится в `.env` и панели, и каждой ноды. Один секрет на пару.
- Для multi-node deployment'а каждая нода может иметь свой `NODE_SECRET` — панель привязывает к node-record'у при регистрации (стандартный Remnawave-флоу).

## Интеграция с xray-runtime ноды

Когда backend вызывает `provision-user`, node не только создаёт PAK на провайдере, но и должен добавить пользователя в живой xray-config. Делается через `handler.service.ts:169-187`:

```typescript
xtlsApi.handler.addTrojanUser({
  tag: inboundTag,
  username: userId,       // используется как fedarisha user.id
  password: '',           // пустой, fedarisha не использует
  level: 0,
})
```

**Это не опечатка.** Fedarisha переиспользует протобаф `Trojan.Account` как пустой carrier — у него правильный shape для xray-api `addUser`, но xray-runtime игнорирует password для fedarisha-протокола (см. [protocol.md#handshake](protocol.md) — auth идёт через PAK + X25519, не через password).

`removeUser` — стандартный `xtlsApi.handler.removeUser(tag, username)`.
