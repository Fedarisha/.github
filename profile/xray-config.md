# Конфигурация Xray

Поля протокола `fedarisha` в конфиге Xray-core-fedarisha. Парсер находится в [`infra/conf/fedarisha.go`](https://github.com/Fedarisha/Xray-core-fedarisha/blob/main/infra/conf/fedarisha.go).

## Inbound (нода)

```jsonc
{
  "tag": "fedarisha-eu",
  "protocol": "fedarisha",
  "settings": {
    "storage": { … },      // обязательно
    "tuning":  { … },      // необязательно
    "webhook": { … },      // необязательно
    "clients": [ … ],      // кого пускать
    "userLevel": 0
  }
}
```

`listen`, `port` и `streamSettings` не используются: инбаунд не открывает сетевой порт, кроме webhook-сервера.

### `clients`

```json
"clients": [ { "id": "alice", "email": "alice", "level": 0 } ]
```

- `id` — имя папки пользователя в бакете (`<prefix>/<id>/…`). Нода принимает сессию, только если первый сегмент пути после `prefix` совпадает с `id` одного из клиентов.
- `email` — имя пользователя для логов, статистики и правил маршрутизации (`user` в routing). По умолчанию равно `id`.
- `level` — уровень политики. Если 0, берётся `userLevel`.

Пустой `clients` означает, что сессии не принимаются вообще. Режима «пускать всех» нет.

Пользователей можно добавлять и удалять на лету через API Xray (`AlterInbound`). В этом случае ключом служит `email`, и он должен совпадать с именем папки. При удалении пользователя его открытые сессии закрываются сразу.

В панели `clients` заполняет backend, в конфиг-профиле этот блок не указывают.

### `storage`

| Поле | Описание |
| --- | --- |
| `type` | `s3` или `local`, регистр не важен. Если не задан: `s3` при заданном `bucket`, иначе `local` при заданном `localDir`. |
| `bucket` | Имя бакета. Обязательно для `s3`. |
| `endpoint` | URL S3 со схемой, например `https://hb.ru-msk.vkcloud-storage.ru`. Если задан, запросы идут в path-style. Если не задан, используется AWS. |
| `region` | Регион для подписи запросов. По умолчанию `us-east-1`. |
| `prefix` | Базовый префикс в бакете. У инбаунда это корень, под которым лежат папки пользователей. |
| `sessionsDir` | Имя папки сессий внутри папки пользователя. По умолчанию `sessions`. Менять не стоит: webhook работает только с этим именем. |
| `accessKey`, `secretKey` | Ключи S3. Инбаунду нужны права на LIST, GET, PUT и DELETE под `prefix`, HeadBucket и настройку lifecycle (без неё правило очистки не ставится, но инбаунд работает). Outbound достаточно прав на объекты в своей папке. |
| `localDir` | Каталог на диске для `type: local`. Нужен только для отладки на одной машине. |

При старте инбаунд выполняет HeadBucket. Если он не проходит, инбаунд не запускается.

Поля `authType`, `masterServiceUserId`, `iam` и `pathStyle` Xray игнорирует. Их читает нода Remnawave, см. [storage-providers.md](storage-providers.md).

### `tuning`

Все поля — целые числа. 0 или отсутствие поля означает значение по умолчанию.

| Поле | По умолчанию | Что делает |
| --- | --- | --- |
| `writeIntervalMs` | 20 | Период сброса буфера записи в S3. |
| `maxFileSizeBytes` | 2097152 | Максимальный размер одного файла данных. Уменьшать не стоит: в замерах 64 KiB давали скорость в 3–4 раза ниже, чем 2 MiB. |
| `idleTimeoutSec` | 300 | Закрыть сессию, если столько секунд не приходило данных. |
| `pollIntervalMs` | 500 | Период сканирования бакета нодой в поиске новых сессий. С webhook принудительно 10 000. На задержку внутри сессии не влияет. |

Задержки опроса внутри сессии зашиты в код, см. [protocol.md](protocol.md#запись-и-чтение).

### `webhook`

| Поле | По умолчанию | Описание |
| --- | --- | --- |
| `enabled` | `false` | Включить приём уведомлений от S3. |
| `listen` | — | Адрес HTTP-сервера, например `":80"`. Обязательно. |
| `publicUrl` | — | Абсолютный URL, по которому S3 будет слать уведомления. Путь из него используется как путь обработчика (пустой путь означает `/webhook`). Обязательно. |
| `autoSetup` | `true` | Зарегистрировать уведомление в бакете при старте. Работает с VK Cloud. |
| `tlsCert`, `tlsKey` | — | Пути к PEM-файлам для HTTPS. Задаются вместе. |

Webhook требует `type: s3`. Ошибки в этом блоке не дают инбаунду стартовать. Панель подставляет значения по умолчанию, если блок `webhook` присутствует, см. [panel-integration.md](panel-integration.md#значения-webhook-по-умолчанию).

## Outbound (клиент)

```json
{
  "tag": "proxy",
  "protocol": "fedarisha",
  "settings": {
    "storage": {
      "type": "s3",
      "bucket": "my-bucket",
      "endpoint": "https://hb.ru-msk.vkcloud-storage.ru",
      "region": "ru-msk",
      "prefix": "fed/alice/",
      "accessKey": "…",
      "secretKey": "…"
    },
    "tuning": { "idleTimeoutSec": 300 }
  }
}
```

Поля `storage` и `tuning` те же, что у инбаунда. Отличие одно: `prefix` клиента — это папка пользователя целиком, `<prefix инбаунда>/<id>/`. Поддерживаются назначения TCP и UDP.

## Минимальный пример без панели

Нужен бакет и ключ с доступом к нему. Для одного-двух пользователей можно дать клиенту тот же ключ, что и ноде. Для изоляции пользователей друг от друга выпустите каждому ключ с доступом только к его папке средствами провайдера.

Нода:

```json
{
  "inbounds": [
    {
      "tag": "fed-in",
      "protocol": "fedarisha",
      "settings": {
        "storage": {
          "type": "s3",
          "bucket": "my-bucket",
          "endpoint": "https://hb.ru-msk.vkcloud-storage.ru",
          "region": "ru-msk",
          "prefix": "fed",
          "accessKey": "…",
          "secretKey": "…"
        },
        "clients": [ { "id": "alice" } ]
      }
    }
  ],
  "outbounds": [ { "protocol": "freedom" } ]
}
```

Клиент: локальный SOCKS-прокси и outbound из предыдущего раздела с `"prefix": "fed/alice/"`.

```json
{
  "inbounds": [
    { "port": 1080, "listen": "127.0.0.1", "protocol": "socks", "settings": { "udp": true } }
  ],
  "outbounds": [ { "tag": "proxy", "protocol": "fedarisha", "settings": { "storage": { … } } } ]
}
```

Запуск: `xray run -c config.json` на обеих сторонах. Нода ищет новые сессии раз в 500 мс, поэтому первое соединение устанавливается за время порядка секунды.
