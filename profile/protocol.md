# Wire-протокол fedarisha-транспорта

Если совсем коротко: **клиент и нода не открывают сокет — они оставляют друг другу файлы в общей папке S3**. Этот документ — про то, какие именно файлы, как они шифруются и в каком порядке появляются.

Исходник всех деталей — [`Xray-core-fedarisha/proxy/fedarisha/`](https://github.com/Fedarisha/Xray-core-fedarisha/tree/main/proxy/fedarisha).

## Анатомия сессии в бакете

```
<bucket>/
├── <basePrefix>/                                    ← settings.storage.prefix
│   ├── <userId-1>/                                  ← PAK ограничен этим префиксом
│   │   └── sessions/                                ← settings.storage.sessionsDir или "sessions"
│   │       ├── <sessionId-aaaa>/                    ← одна сессия = одна папка
│   │       │   ├── c_hello                          ← handshake-объект (client→server)
│   │       │   ├── s_ack                            ← handshake-ответ  (server→client)
│   │       │   ├── c_00000000  c_00000001  ...      ← кадры client→server
│   │       │   └── s_00000000  s_00000001  ...      ← кадры server→client
│   │       └── <sessionId-bbbb>/...
│   └── <userId-2>/...
└── .fedarisha-pak-state/                            ← только для Selectel-провайдера
    └── <basePrefix>.json
```

Соглашения о именах:

- `sessionId` — 16 случайных байт в hex (32 символа).
- Кадры именуются `{c_|s_}{08x}` — direction-prefix + zero-padded hex sequence number.
- Клиент пишет только `c_*`, нода — только `s_*`. Перекрытий нет.

S3 lifecycle-rule `Expiration.Days: 1` ставится `SetupLifecycle` на старте инбаунда — защита от мёртвых сессий, которые никто не закрыл явно.

## Handshake: X25519 поверх трёх объектов

Без сертификатов, без CA. Аутентификация — через сам факт, что у клиента есть PAK с доступом к папке: если ты можешь писать в `<bucket>/<basePrefix>/<userId>/sessions/...`, ты — этот юзер.

### c_hello — клиент кричит «привет»

Клиент:

1. Генерирует `sessionId` (16 байт rand).
2. Создаёт keypair X25519 (`clientPrivate`, `clientPublic`).
3. Кладёт в S3 объект `<sessions>/<sessionId>/c_hello` с телом:
   ```
   [sessionId (16 байт) || clientPublic (32 байта)]
   ```

С точки зрения S3 это обычный PUT 48-байтового файла. Никакого ответа от ноды клиент пока не получил — он сейчас будет опрашивать `s_ack`.

### s_ack — нода отвечает

Нода узнаёт о новой сессии одним из двух способов:

- **С webhook'ом:** S3 шлёт нотификацию `s3:ObjectCreated:Put`, регистрация активной сессии срабатывает мгновенно (см. ниже про webhook fast-path).
- **Без webhook'а:** listener делает `LIST` сессионных папок с интервалом `pollIntervalMs` (по умолчанию 500 мс если webhook есть как fallback, 10 секунд если webhook'а нет вообще — нюанс: значения подобраны так, чтобы не молотить S3, когда есть основной канал нотификаций).

Дальше нода:

1. Читает `c_hello`, проверяет `IsUserAllowed(userPrefix)` (server gate — закрывает race window между revoke и реальным удалением PAK).
2. Генерирует свой keypair X25519, считает `sharedKey = X25519(serverPrivate, clientPublic)`.
3. Выводит ключ AES через HKDF-SHA256:
   ```
   sessionKey = HKDF-SHA256(
       IKM  = sharedKey,
       salt = sessionId,
       info = "fedarisha-e2e-v1",
       len  = 32
   )
   ```
4. Пишет `<sessions>/<sessionId>/s_ack` с телом — `serverPublic` (32 байта).
5. Удаляет `c_hello`.

### Клиент подтверждает

Клиент опрашивает `s_ack` до 60 секунд. Когда видит:

1. Получает `serverPublic`, считает тот же `sharedKey` и `sessionKey`.
2. Удаляет `s_ack`.
3. Поднимает yamux-мультиплекс поверх encrypted stream.

С этого момента обе стороны пишут/читают кадры `c_<seq>` / `s_<seq>` со sequence от 0. Папка сессии больше не содержит handshake-объектов — только пронумерованные кадры.

## Один кадр

Каждый файл `c_<seq>` / `s_<seq>` — это один AES-GCM ciphertext:

```
ciphertext = AES-256-GCM-Encrypt(
    key       = sessionKey,
    nonce     = MakeNonce(direction, seq),
    aad       = sessionId,
    plaintext = framePayload
)
```

`MakeNonce` строит 12-байтовый nonce из:

```
nonce[0:2]  = ASCII direction prefix ("c_" = 0x63 0x5f, "s_" = 0x73 0x5f)
nonce[2:4]  = 0x00 0x00 (padding)
nonce[4:12] = big-endian uint64(seq)
```

Это даёт уникальность nonce до 2⁶⁴ кадров в направление (практически бесконечность). `aad = sessionId` обеспечивает, что кадр другой сессии не пройдёт authentication tag — даже если кто-то его подсунет.

### Что лежит внутри ciphertext

После расшифровки:

```
plaintext[0]  = compression flag   (0x00 = raw, 0x01 = deflate)
plaintext[1:] = (raw или deflate-сжатые) yamux-данные
```

Encoder пробует deflate с `BestSpeed` (level 1). Если сжатый меньше исходного — флаг `0x01` + сжатые данные, иначе `0x00` + raw. Флаг — **байт внутри ciphertext**, имя файла от компрессии не зависит.

Yamux-data — стандартный yamux-стрим: handshake → frames → keepalive (60s). Внутри yamux-стрима каждое соединение начинается с **target header** (proxy-destination):

```
[len_or_udp_flag uint16 BE]   ← бит 15 = 1 → UDP, иначе TCP
[host bytes — N байт]
[port uint16 BE]
```

Дальше — обычный bidirectional bytestream.

### UDP-обёртка

Поверх yamux-стрима для UDP-сессий каждый пакет фреймится:

```
[metaLen uint16 BE]
[metadata: dest address+port (xray AddressPort encoding) — если metaLen > 0]
[payloadLen uint16 BE]
[payload — N байт]
```

Это позволяет stateless-проксировать UDP с пересылкой адреса назначения в каждом пакете.

## Как читаются кадры (важно: не LIST)

Conn'ы на обеих сторонах знают свой текущий `readSeq` и **запрашивают конкретный файл напрямую** (`GET <sessDir>/c_00000007`), а не сканируют папку через LIST. Это спасает от того, что LIST в S3 — eventually consistent и платный по другому тарифу.

Чтобы скрыть latency:

- Параллельно запускаются GET'ы для `seq`, `seq+1`, …, `seq+ahead` (ahead = 2 в idle, 4 в активной фазе).
- Результаты складываются в `prefetchCache: map[uint64][]byte`.
- При получении ожидаемого `seq` — берётся из кеша. Out-of-order ответы дождутся своего номера.
- Дырки (404 на `seq+1` при наличии `seq+2`) останавливают consume — данные за gap'ом подождут, пока gap закроется.

После того как кадр успешно расшифрован и положен в `readBuf`, его файл **удаляется немедленно** — в detached-горутине с собственным 10-секундным контекстом:

```go
go func(p string) {
    dctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    _ = c.store.Delete(dctx, p)
}(s.path)
```

Это даёт бакету не разрастаться даже на долгих сессиях. Если delete по какой-то причине не успел — на `conn.Close()` отработает `cleanupSession`, которая через `BatchDelete` (или per-file fallback) сметёт всё, что осталось в папке сессии.

## Адаптивный polling

Когда новых кадров нет, частота poll'а зависит от того, как давно был последний полезный фрейм:

| Время с последнего получения | Интервал между fetchNext |
| --- | --- |
| `< 5s` | 20 мс — активный transfer |
| `5–30s` | 100 мс — возможно, страница ещё догружается |
| `> 30s` | 500 мс — truly idle |

Если включён webhook — pollLoop вообще блокируется на канале нотификаций, и `select` срабатывает либо по сигналу, либо по 30-секундному safety-таймауту (страховка от потерянных нотификаций). В типичном S3-инбаунде с webhook'ом 99% wakeup'ов идут от нотификаций, не от таймера.

## Tuning: параметры конфига

Четыре параметра живут в `settings.tuning` инбаунда. Полные дефолты и формат — [inbound-config.md](inbound-config.md), здесь — смысл в терминах протокола.

| Параметр | Где применяется | Влияние |
| --- | --- | --- |
| `pollIntervalMs` | `watchLoop` в listener.go | Как часто listener делает LIST по `sessionsDir`, чтобы заметить новую папку сессии (без webhook'а — дефолт 500 мс; с webhook'ом принудительно 10 с, значение из tuning игнорируется). На pollLoop уже установленной conn'ы **не влияет** — там адаптивные тайеры 20/100/500 мс зашиты в код (поле `c.pollInterval` хранится, но в hot-path не читается). |
| `writeIntervalMs` | `writeLoop` в conn.go | Как часто writer флашит accumulator в S3. Меньше → ниже latency, но больше PUT'ов и мельче объекты. |
| `idleTimeoutSec` | `idleWatcher` в conn.go | Сколько секунд без полученных данных до закрытия. Yamux keepalive (60s) ресетит таймер. |
| `maxFileSizeBytes` | flush в conn.go | Лимит одного кадра. Превышение → разбивка на N PUT'ов размером ≤ лимита. |

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
2. Из key вытаскивает `userPrefix` и `sessionId`. Если сессия зарегистрирована (`Register(sessionId)` сделан в listener'е при handshake'е) — сигналит её каналу.
3. `Conn.pollLoop` пробуждается, делает GET вне расписания.

Без webhook'а латентность доставки нового кадра ≈ половина адаптивного тайера conn'ы (~10–250 мс на активной сессии). С webhook'ом — `<100 мс` end-to-end. Webhook **не отменяет poll'а** — 30-секундный fallback страхует от потерянных нотификаций. Tuning-параметр `pollIntervalMs` влияет на задержку *обнаружения новой сессии* listener'ом, но не на latency внутри уже установленной сессии.

VK Cloud S3 валидирует подписку HMAC-цепочкой `sig = hmac(url, hmac(arn, hmac(timestamp, token)))` — `WebhookHub` отвечает этим значением автоматически (см. [`storage/s3/webhook.go`](https://github.com/Fedarisha/Xray-core-fedarisha/blob/main/proxy/fedarisha/storage/s3/webhook.go)).

### Shared listener для нескольких инбаундов

Несколько fedarisha-инбаундов на одной ноде могут шарить один HTTP-листенер (`webhook_registry.go`):

- Один `http.Server` на `host:port`, разные `publicUrl`-paths мультиплексируются.
- Конфликт TLS-настроек на одном `listen` → fatal на старте (`mismatched TLS for listen`).
- Конфликт `publicUrl` на одном `listen` → тоже fatal.

То есть на одной ноде можно поднять `fedarisha-eu`, `fedarisha-us`, `fedarisha-ru` инбаунды и обойтись одним `:80` listener'ом.

## Lifecycle и cleanup

| Объект | Когда удаляется | Кем |
| --- | --- | --- |
| Прочитанный кадр | Сразу после consume, async (10s context per file) | `pollLoop` в conn.go |
| Все кадры сессии + сама папка | На `conn.Close()` | `cleanupSession` через `BatchDelete` (или per-file fallback) |
| Брошенная сессия (клиент исчез без Close) | Не позже 24h | S3 lifecycle rule `Expiration.Days: 1`, поставленный `SetupLifecycle` |
| Brokered crash recovery | На рестарте инбаунда — нет (полагаемся на lifecycle) | — |

**Quirk:** `BatchDelete` через `DeleteObjects` поддерживают не все S3-совместимые сторы (Garage до v1.0 не поддерживал). Без него `Close` падает на per-file delete и может не успеть в дедлайн на сессии с тысячами кадров. Если используете не AWS/VK/Selectel/MinIO — проверьте поддержку.

Константа `CleanupAge = 30 * time.Second` в `protocol.go` объявлена, но **в hot-path не используется** — рудимент старой реализации. Реальное удаление — немедленное.

## Storage type — что понимает xray

`infra/conf/fedarisha.go` принимает в `settings.storage.type`:

- `"s3"` или пусто (если задан `bucket`) — S3 backend.
- `"local"` или пусто (если задан `localDir`) — локальная файловая система (для отладки).
- `"vkcloud-pak"`, `"selectel-iam"`, `"static"` — алиасы на `"s3"` (нода-провайдеры коллапсируются здесь, чтобы конфиг можно было копировать между inbound и outbound без правок).

Если ни одно условие не выполнено — fatal на парсинге.

**На клиенте бэкенд всегда подставляет `type: "s3"`** ([почему](subscription-flow.md#почему-type-s3)) — клиентский xray-core-fedarisha PAK-провайдеров не знает и не должен знать.
