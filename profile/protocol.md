# Wire-протокол fedarisha-транспорта

Что лежит в бакете, как клиент и нода о нём договариваются. Исходник — [`Xray-core-fedarisha/proxy/fedarisha/`](https://github.com/Fedarisha/Xray-core-fedarisha/tree/main/proxy/fedarisha).

## Раскладка объектов в бакете

```
<bucket>/
├── <basePrefix>/                                    ← из settings.storage.prefix
│   ├── <userId-1>/                                  ← per-user PAK ограничен этим префиксом
│   │   └── sessions/                                ← settings.storage.sessionsDir или "sessions"
│   │       ├── <sessionId-aaaa>/                    ← одна сессия = одна папка
│   │       │   ├── c_hello                          ← handshake-объект (client→server)
│   │       │   ├── s_ack                            ← handshake-ответ (server→client)
│   │       │   ├── c_00000000   c_00000001   ...    ← кадры client→server
│   │       │   └── s_00000000   s_00000001   ...    ← кадры server→client
│   │       └── <sessionId-bbbb>/...
│   └── <userId-2>/...
└── .fedarisha-pak-state/                            ← только для Selectel-провайдера
    └── <basePrefix>.json
```

- `sessionId` — 16 случайных байт в hex (32 символа).
- Имена кадров: `{c_|s_}{08x}` — direction-prefix + zero-padded hex sequence number, по одному на каждый направление.
- Клиент пишет только `c_*`, нода — только `s_*`. Перекрытий нет.
- Lifecycle-правило (для S3-провайдеров) автоматически expires объекты старше 1 дня — защита от мёртвых сессий.

## Handshake

Двусторонний X25519 + AES-256-GCM. Без сертификатов и без CA — auth через знание PAK-ключей (доступ к бакету = доказательство пользователя).

### Шаг 1: c_hello (клиент → S3)

Клиент:

1. Генерирует `sessionId` (16 байт rand).
2. Создаёт keypair X25519 (`clientPrivate`, `clientPublic`).
3. Кладёт в S3 объект `<sessions>/<sessionId>/c_hello` с телом:
   ```
   [sessionId (16 байт) || clientPublic (32 байта)]
   ```

### Шаг 2: s_ack (нода → S3)

Нода (либо по webhook'у, либо на следующем `pollIntervalMs`):

1. Видит новую sessions-папку, читает `c_hello`.
2. Проверяет `IsUserAllowed(userPrefix)` (server gate, см. [architecture.md#уровни-изоляции](architecture.md)).
3. Генерирует keypair X25519, считает shared secret `sharedKey = X25519(serverPrivate, clientPublic)`.
4. Деривит ключ AES через HKDF-SHA256:
   ```
   sessionKey = HKDF-SHA256(
       IKM  = sharedKey,
       salt = sessionId,
       info = "fedarisha-e2e-v1",
       len  = 32
   )
   ```
5. Пишет `<sessions>/<sessionId>/s_ack` с телом `serverPublic` (32 байта).
6. Удаляет `c_hello`.

### Шаг 3: клиент подтверждает

Клиент опрашивает `s_ack` до 60 секунд:

1. Получает `serverPublic`, считает тот же `sharedKey` и `sessionKey`.
2. Удаляет `s_ack`.
3. Поднимает yamux-мультиплекс поверх encrypted stream.

С этого момента обе стороны пишут/читают кадры `c_<seq>` / `s_<seq>` со sequence от 0.

## Кадр (encrypted chunk)

Каждый файл `c_<seq>` / `s_<seq>` — это один AES-GCM ciphertext:

```
ciphertext = AES-256-GCM-Encrypt(
    key   = sessionKey,
    nonce = MakeNonce(direction, seq),
    aad   = sessionId,
    plaintext = framePayload
)
```

`MakeNonce` (12 байт):

```
nonce[0:2]  = ASCII direction prefix ("c_" = 0x63 0x5f, "s_" = 0x73 0x5f)
nonce[2:4]  = 0x00 0x00 (padding)
nonce[4:12] = big-endian uint64(seq)
```

Это гарантирует уникальность nonce при ограничении на ≤ 2^64 кадров в направление (для практики бесконечность).

`aad = sessionId` обеспечивает, что кадр другой сессии не пройдёт authentication tag — даже если кто-то его подсунет.

## Frame payload (внутри ciphertext)

После расшифровки получается:

```
plaintext[0] = compression flag (0x00 = raw, 0x01 = deflate)
plaintext[1:] = (raw или deflate-compressed) yamux-data
```

Encoder пробует deflate с `BestSpeed` (level 1). Если сжатый меньше исходного — флаг 0x01 + сжатые данные, иначе 0x00 + raw.

Yamux-data — стандартный yamux-стрим: handshake → frames → keepalive (60s). Внутри yamux-стрима каждое соединение начинается с **target header** (proxy-destination):

```
[len_or_udp_flag uint16 BE]   ← биту 15 = 1 → UDP, иначе TCP
[host bytes — N байт]
[port uint16 BE]
```

Дальше — обычный bidirectional bytestream.

## UDP encapsulation

Поверх yamux-стрима для UDP-сессий каждый пакет фреймится:

```
[metaLen uint16 BE]
[metadata: dest address+port (xray AddressPort encoding) — если metaLen > 0]
[payloadLen uint16 BE]
[payload — N байт]
```

Это позволяет stateless-проксировать UDP с пересылкой адреса назначения в каждом пакете.

## Tuning (что управляется конфигом)

Все четыре параметра живут в `settings.tuning` инбаунда. Полные значения и дефолты — [inbound-config.md](inbound-config.md), здесь — их смысл в терминах протокола.

| Параметр | Где применяется | Влияние |
| --- | --- | --- |
| `pollIntervalMs` | `pollLoop` в conn.go | Базовый интервал LIST-запроса при отсутствии webhook'а или после misses. Адаптивный: при активности `<5s` → 20ms; `5-30s` → 100ms; `idle >30s` → 500ms. |
| `writeIntervalMs` | `writeLoop` в conn.go | Как часто writer флашит accumulator в S3. Меньше → ниже latency, но больше PUT'ов и мельче объекты. |
| `idleTimeoutSec` | `idleWatcher` в conn.go | Сколько секунд без полученных данных до закрытия. Yamux keepalive (60s) ресетит таймер. |
| `maxFileSizeBytes` | flush в conn.go | Лимит одного кадра. Превышение → разбивка на N PUT'ов размером ≤ лимита. |

**`CleanupAge` = 30 секунд, hardcoded в `proxy/fedarisha/transport/protocol.go`.** Объекты, помеченные как обработанные, удаляются не сразу — даётся 30 секунд на возможный re-read из prefetch cache. Не настраивается из конфига.

## Prefetch cache

Чтобы скрыть latency S3-LIST'а, conn.go проактивно тянет 2–4 кадра вперёд:

- Параллельно запускаются GET'ы для `seq+1, seq+2, ..., seq+ahead` (количество = 2-4 в зависимости от активности).
- Результаты складываются в `prefetchCache: map[uint64][]byte`.
- При получении ожидаемого `seq` — берётся из кеша. Out-of-order GET'ы дождутся своего номера.
- Дырки (404 на `seq+1` при наличии `seq+2`) останавливают consume — данные за gap'ом флашатся когда gap закроется.

Это даёт сущ. снижение latency при сохранении ordering — стандартный шаблон storage-backed протоколов.

## Webhook fast-path

Если в `settings.webhook` сконфигурен HTTP-listener, нода регистрирует у S3-провайдера notification на `s3:ObjectCreated:Put`. При получении нотификации:

1. `WebhookHub.ServeHTTP` парсит JSON:
   ```jsonc
   {
     "Records": [{
       "s3": {
         "bucket": {"name": "..."},
         "object": {"key": "users/main/123/sessions/abcd/c_00000007", "size": 1234}
       }
     }]
   }
   ```
2. Из key вытаскивает `userPrefix` и `sessionId`. Если сессия зарегистрирована (`Register(sessionId)` сделан в listener'е) — сигналит каналу.
3. Conn.pollLoop пробуждается, делает GET вне расписания.

Без webhook'а латентность доставки = `(pollInterval / 2)` в среднем (50–250ms). С webhook'ом — `<100ms` end-to-end. Webhook **не отменяет poll'а** — 30-секундный fallback страхует от потерянных нотификаций.

VK Cloud S3 валидирует подписку HMAC-цепочкой `sig = hmac(url, hmac(arn, hmac(timestamp, token)))` — `WebhookHub` отвечает этим значением автоматически (см. [`storage/s3/webhook.go`](https://github.com/Fedarisha/Xray-core-fedarisha/blob/main/proxy/fedarisha/storage/s3/webhook.go)).

## Shared webhook listener

Несколько fedarisha-инбаундов на одной ноде могут шарить один HTTP-листенер. `webhook_registry.go`:

- Один `http.Server` на `host:port`, разные `publicUrl`-paths мультиплексируются на один listener.
- Конфликт TLS-настроек на одном `listen` — fatal на старте (`mismatched TLS for listen`).
- Конфликт `publicUrl` на одном `listen` — тоже fatal.

Это позволяет на одной ноде поднять `fedarisha-eu`, `fedarisha-us`, `fedarisha-ru` инбаунды и обойтись одним `:80` listener'ом.

## Lifecycle и cleanup

| Что | Когда удаляется | Кем |
| --- | --- | --- |
| Прочитанный кадр | Через `CleanupAge` (30s) после consume | conn.go async delete (10s timeout per call) |
| Все кадры сессии | На `conn.Close()` | conn.go bulk via `BatchDelete` (≤1000 ключей за вызов) |
| Брошенная сессия | Не позже 24h | S3 lifecycle rule `Expiration.Days: 1` (поставлена `SetupLifecycle` на старте инбаунда) |
| Brokered crash recovery | На рестарте инбаунда — нет (полагаемся на lifecycle) | — |

**Quirk:** `BatchDelete` через `DeleteObjects` поддерживают не все S3-совместимые сторы (Garage до v1.0 не поддерживал). В этом случае conn падает на per-file Delete — `Close` может не успеть в 60s deadline на сессии с тысячами кадров. Если используете не AWS/VK/Selectel/MinIO — проверьте поддержку.

## Storage type — что понимает xray

`infra/conf/fedarisha.go` принимает в `settings.storage.type`:

- `"s3"` или пусто (если задан `bucket`) — S3 backend.
- `"local"` или пусто (если задан `localDir`) — локальная файловая система (для отладки).
- `"vkcloud-pak"`, `"selectel-iam"`, `"static"` — алиасы на `"s3"` (нода-провайдеры коллапсируются здесь, чтобы конфиг можно было копировать между inbound и outbound без правок).

Если ни одно условие не выполнено — fatal на парсинге.

**На клиенте backend подставляет `type: "s3"` (см. [subscription-flow.md](subscription-flow.md))** — клиентский xray-core-fedarisha PAK-провайдеров не знает и не должен знать.
