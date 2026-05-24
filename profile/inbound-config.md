# fedarisha-инбаунд: полная схема

JSON-блок, который панель кладёт в xray config как один из элементов `inbounds[]`. Парсер — [`Xray-core-fedarisha/infra/conf/fedarisha.go`](https://github.com/Fedarisha/Xray-core-fedarisha/blob/main/infra/conf/fedarisha.go), проверка/исполнение — [`proxy/fedarisha/{inbound,outbound,webhook_registry}.go`](https://github.com/Fedarisha/Xray-core-fedarisha/tree/main/proxy/fedarisha).

## Скелет

```jsonc
{
  "tag": "fedarisha-eu",
  "protocol": "fedarisha",
  "settings": {
    "storage":  { /* см. storage-providers.md */ },
    "tuning":   { /* опц. — таймауты транспорта */ },
    "webhook":  { /* опц. — push-уведомления S3 → инбаунд */ },
    "clients":  [],                // оставлять пустым: панель заполняет на provision
    "userLevel": 0                 // опц., применяется если у конкретного client нет своего level
  }
  // streamSettings НЕ задаются: у fedarisha нет network-endpoint'а
}
```

`streamSettings`, `sniffing`, `listen`, `port` для fedarisha-инбаунда игнорируются — транспорт не слушает TCP/UDP, данные ходят через S3-бакет.

## `settings.storage` (обязательно)

Полная схема и описание провайдеров — в [storage-providers.md](storage-providers.md). Краткое напоминание полей:

| Поле | Тип | Обяз. | Описание |
| --- | --- | --- | --- |
| `type` | string | да | `vkcloud-pak` \| `selectel-iam` \| `static` — выбирает PAK-провайдера на node-стороне. Дефолта нет. |
| `bucket` | string | да | Имя S3-бакета (выделить отдельно под fedarisha). |
| `endpoint` | string | да | S3-endpoint без схемы (`hb.ru-msk.vkcloud-storage.ru`). |
| `region` | string | нет | Регион для подписи запросов; пустая строка работает на большинстве S3. |
| `prefix` | string | нет | Base-prefix внутри бакета (используется как корень для per-user префиксов). |
| `accessKey` / `secretKey` | string | да | Master-ключи (vkcloud-pak / static — рабочие; selectel — только для `PutBucketPolicy`). |
| `iam.*` | object | для selectel-iam | Учётка IAM-админа Selectel — см. [storage-providers.md#selectel](storage-providers.md). |
| `pathStyle` | bool | нет | Только для `static`; принудительно path-style URL (MinIO/Garage без DNS). |
| `sessionsDir` | string | нет | Дополнительный поддиректорий для активных сессий внутри `prefix`. Дефолт — корень `prefix`. |
| `localDir` | string | нет | Альтернатива S3 — локальная директория (`type` пустой + `localDir` задан). Только для отладки на одной машине. |

## `settings.tuning` (опц.)

Таймауты transport-слоя. Все поля — uint32, нулевое значение означает «использовать дефолт».

| Поле | Дефолт | Что делает |
| --- | --- | --- |
| `pollIntervalMs` | 100 | Как часто читатель опрашивает S3 на новые файлы сессии. Меньше → ниже latency, больше → меньше LIST/GET-запросов. |
| `writeIntervalMs` | 20 | Как часто writer флашит накопленные данные в S3. Меньше → ниже latency, больше → крупнее объекты и меньше PUT'ов. |
| `idleTimeoutSec` | 300 | Сессия закрывается, если не было обмена столько секунд. |
| `maxFileSizeBytes` | 2 097 152 (2 MiB) | Лимит одного файла-чанка в S3. Превышение → автоматическая ротация на следующий sequence-файл. |

```jsonc
"tuning": {
  "pollIntervalMs": 200,
  "writeIntervalMs": 50,
  "idleTimeoutSec": 600,
  "maxFileSizeBytes": 4194304
}
```

Гарбадж-коллектор удаляет файлы старше 30 секунд (`CleanupAge`, не настраивается).

## `settings.webhook` (опц.)

S3 умеет слать HTTP-нотификации на создание объекта — это позволяет инбаунду реагировать на новый чанк мгновенно, не дожидаясь `pollIntervalMs`. Поддерживается только S3-storage (vkcloud-pak / selectel-iam / static), для `localDir` блок игнорируется.

| Поле | Тип | Дефолт (если backend сам подставит — см. ниже) | Описание |
| --- | --- | --- | --- |
| `enabled` | bool | `true` | Полный switch on/off. Если `false` — все остальные поля игнорятся. |
| `listen` | string | `":80"` | `host:port` для HTTP-сервера webhook'а внутри node-контейнера. |
| `publicUrl` | string | `http://<nodeAddress>/webhook` | Полный URL, который зарегистрируется в S3-источнике; должен быть достижим от S3-провайдера. |
| `autoSetup` | bool | `true` | Если `true` — node сам зовёт S3 API и регистрирует `publicUrl`. Если `false` — настройка вручную (например, через консоль VK Cloud). |
| `tlsCert` / `tlsKey` | string | `""` | Пути к PEM-файлам внутри контейнера для HTTPS. Должны задаваться вместе. |

```jsonc
"webhook": {
  "enabled": true,
  "listen": ":8080",
  "publicUrl": "https://node-eu.example.com/webhook",
  "autoSetup": true
}
```

**Backend подставляет дефолты автоматически**, если `webhook` блок присутствует в инбаунде с одним из S3-провайдеров: `enabled=true`, `listen=":80"`, `publicUrl=http://<nodeAddress>/webhook`, `autoSetup=true` ([`backend/src/common/utils/apply-fedarisha-webhook-defaults.ts`](https://github.com/Fedarisha/backend/blob/main/src/common/utils/apply-fedarisha-webhook-defaults.ts)). Чтобы получить дефолты — оставьте `"webhook": {}`. Чтобы webhook полностью отключить — уберите ключ или укажите `"enabled": false`.

Ошибки конфигурации webhook (несовпадающие `publicUrl` на одном `listen`, конфликт `tlsCert`/`tlsKey`, пустой `publicUrl`) роняют инбаунд при старте — проверяйте логи xray.

## `settings.clients`

Список панель-пользователей, которым разрешён вход. Заполняется автоматически backend'ом на основе подписки — **в xray-конфиге, который кладёте в config-profile панели, держите пустой массив или вовсе не указывайте**.

```jsonc
"clients": [
  { "id": "<uuid>", "email": "user@example.com", "level": 0 }
]
```

`id` — uuid пользователя из панели, `email` — для трейсинга в логах, `level` — переопределяет `settings.userLevel`. Уровни используются Xray для policy-bind (`policy.levels`).

## `settings.userLevel`

uint32, фолбэк-level для всех `clients` без своего `level`. Дефолт `0`.

## Что фактически уходит клиенту

Outbound, который backend рендерит в подписку, **не повторяет inbound один-в-один**:

- `storage.type` всегда переписывается на `"s3"` (клиентский xray не знает про `vkcloud-pak`/`selectel-iam`/`static` — это node-side понятия для выбора PAK-провайдера; см. [`backend/src/modules/fedarisha-provisioning/fedarisha-subscription.service.ts`](https://github.com/Fedarisha/backend/blob/main/src/modules/fedarisha-provisioning/fedarisha-subscription.service.ts));
- `accessKey`/`secretKey` заменяются на per-user PAK, выданный node'ом;
- `prefix` заменяется на per-user префикс (`<basePrefix>/<userUuid>/`);
- `tuning` пробрасывается как есть;
- `webhook` НЕ передаётся (это серверная фича);
- `clients`/`userLevel` НЕ передаются (на клиенте нет аутентификации пользователей).
