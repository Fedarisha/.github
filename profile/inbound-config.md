# fedarisha-инбаунд: полная схема

Один JSON-элемент массива `inbounds[]` в xray-config панели. Парсер на ноде — [`Xray-core-fedarisha/infra/conf/fedarisha.go`](https://github.com/Fedarisha/Xray-core-fedarisha/blob/main/infra/conf/fedarisha.go), исполнение — [`proxy/fedarisha/{inbound,outbound,webhook_registry}.go`](https://github.com/Fedarisha/Xray-core-fedarisha/tree/main/proxy/fedarisha).

## Из чего состоит

У fedarisha-инбаунда четыре содержательных под-блока:

- **`storage`** — обязательный. Где живёт бакет и какой PAK-провайдер выдаёт ключи. Полная схема — в [storage-providers.md](storage-providers.md).
- **`tuning`** — опциональный. Таймауты transport-слоя.
- **`webhook`** — опциональный. Включает push-нотификации от S3 в инбаунд (без него — polling).
- **`clients`** — оставлять пустым. Backend заполняет на provision.

`streamSettings`, `sniffing`, `listen`, `port` для fedarisha-инбаунда игнорируются — транспорт не открывает сетевой endpoint, данные ходят через файлы в бакете.

## Скелет

```jsonc
{
  "tag": "fedarisha-eu",
  "protocol": "fedarisha",
  "settings": {
    "storage":  { /* см. storage-providers.md */ },
    "tuning":   { /* опц. — таймауты транспорта */ },
    "webhook":  { /* опц. — push-уведомления S3 → инбаунд */ },
    "clients":  [],            // оставлять пустым: панель заполняет на provision
    "userLevel": 0             // опц., fallback level для клиентов без своего
  }
  // streamSettings, sniffing, listen, port — НЕ задавать
}
```

## `settings.storage` (обязательно)

Полная таблица провайдеров, их особенности и примеры — в [storage-providers.md](storage-providers.md). Краткое напоминание полей:

| Поле | Тип | Обяз. | Описание |
| --- | --- | --- | --- |
| `type` | string | да | `vkcloud-pak` \| `selectel-iam` \| `static` — выбирает PAK-провайдера. Дефолта нет. |
| `bucket` | string | да | Имя S3-бакета (выделить отдельно под fedarisha). |
| `endpoint` | string | да | S3-endpoint без схемы (`hb.ru-msk.vkcloud-storage.ru`). |
| `region` | string | нет | Регион для подписи; пустая строка работает на большинстве S3. |
| `prefix` | string | нет | Base-prefix внутри бакета (корень для per-user префиксов). |
| `accessKey` / `secretKey` | string | да | Master-ключи (vkcloud-pak / static — рабочие; selectel — только для `PutBucketPolicy`). |
| `iam.*` | object | для selectel-iam | Учётка IAM-админа Selectel — см. [storage-providers.md](storage-providers.md). |
| `pathStyle` | bool | нет | Только для `static`; принудительно path-style URL (MinIO/Garage без DNS). |
| `sessionsDir` | string | нет | Поддиректорий для активных сессий внутри `prefix`. Дефолт — корень `prefix`. |
| `localDir` | string | нет | Альтернатива S3 — локальная директория (`type` пустой + `localDir` задан). Только для отладки на одной машине. |

## `settings.tuning` (опц.)

Четыре таймаута. Все поля — uint32, нулевое значение → дефолт.

| Поле | Дефолт | Что делает |
| --- | --- | --- |
| `pollIntervalMs` | listener: `500` без webhook'а, `10 000` с webhook'ом | Как часто **listener** проверяет S3 на новые сессии. На скорость чтения уже установленной сессии не влияет — там работают адаптивные тайеры (см. [protocol.md](protocol.md#адаптивный-polling)). |
| `writeIntervalMs` | `20` | Как часто writer флашит accumulator в S3. Меньше → ниже latency, больше → крупнее объекты и меньше PUT'ов. |
| `idleTimeoutSec` | `300` | Сессия закрывается, если не было полученных данных столько секунд. Yamux keepalive (60с) ресетит таймер. |
| `maxFileSizeBytes` | `2 097 152` (2 MiB) | Лимит одного файла-чанка в S3. Превышение → разбивка на N PUT'ов размером ≤ лимита. |

```jsonc
"tuning": {
  "pollIntervalMs": 200,
  "writeIntervalMs": 50,
  "idleTimeoutSec": 600,
  "maxFileSizeBytes": 4194304
}
```

**Cleanup.** Прочитанные кадры удаляются **немедленно** после consume (detached-горутина с собственным 10-секундным контекстом). Брошенные сессии (клиент исчез без `Close`) подметает S3 lifecycle rule `Expiration.Days: 1`, который ставится `SetupLifecycle` на старте инбаунда. Никакого настраиваемого `CleanupAge` нет — константа с таким именем в коде осталась как рудимент старой реализации.

## `settings.webhook` (опц.)

S3 умеет слать HTTP-нотификации на создание объекта. Это позволяет инбаунду реагировать на новую сессию мгновенно, а не ждать следующего `pollIntervalMs`. Поддерживается только S3-storage (vkcloud-pak / selectel-iam / static); для `localDir` блок игнорируется.

| Поле | Тип | Дефолт (если backend подставит — см. ниже) | Описание |
| --- | --- | --- | --- |
| `enabled` | bool | `true` | Полный switch. Если `false` — остальные поля игнорятся. |
| `listen` | string | `":80"` | `host:port` HTTP-сервера webhook'а внутри node-контейнера. |
| `publicUrl` | string | `http://<nodeAddress>/webhook` | URL, который зарегистрируется в S3; должен быть достижим от провайдера. |
| `autoSetup` | bool | `true` | Если `true` — нода сама зовёт S3 API и регистрирует `publicUrl`. Если `false` — настраивайте через консоль провайдера. |
| `tlsCert` / `tlsKey` | string | `""` | Пути к PEM-файлам внутри контейнера для HTTPS. Задаются вместе. |

```jsonc
"webhook": {
  "enabled": true,
  "listen": ":8080",
  "publicUrl": "https://node-eu.example.com/webhook",
  "autoSetup": true
}
```

**Три состояния `webhook` в конфиге панели** (различает [`apply-fedarisha-webhook-defaults.ts`](https://github.com/Fedarisha/backend/blob/main/src/common/utils/apply-fedarisha-webhook-defaults.ts)):

- Ключ `webhook` **отсутствует** → backend ничего не подставляет, webhook отключён.
- `"webhook": {}` → backend заполняет всё дефолтами.
- `"webhook": { "enabled": false }` → отключён явно.

Ошибки конфигурации webhook (несовпадающие `publicUrl` на одном `listen`, конфликт `tlsCert`/`tlsKey`, пустой `publicUrl`) роняют инбаунд при старте — смотрите логи xray.

## `settings.clients` и `settings.userLevel`

`clients` — список панель-юзеров, которым разрешён вход. Заполняется автоматически бэкендом на основе подписки — **в xray-config, который вы кладёте в config-profile, держите пустой массив или вовсе не указывайте**:

```jsonc
"clients": [
  { "id": "<uuid>", "email": "user@example.com", "level": 0 }
]
```

`id` — uuid пользователя из панели, `email` — для трейсинга в логах, `level` — переопределяет `settings.userLevel`. Уровни используются Xray для policy-bind (`policy.levels`).

`userLevel` — uint32, fallback-level для всех `clients` без своего. Дефолт `0`.

## Что фактически уходит клиенту

Outbound в подписке **не повторяет inbound один-в-один**:

- `storage.type` всегда переписывается на `"s3"` (клиентский xray не знает про PAK-провайдеры — это node-side понятие).
- `accessKey`/`secretKey` заменяются на per-user PAK, выданный нодой.
- `prefix` заменяется на per-user префикс (`<basePrefix>/<userId>/`).
- `tuning` пробрасывается как есть.
- `webhook` НЕ передаётся (серверная фича).
- `clients`/`userLevel` НЕ передаются (на клиенте нет аутентификации пользователей).

Подробности рендера — [subscription-flow.md](subscription-flow.md#рендер-подписки-buildoutboundforhost).
