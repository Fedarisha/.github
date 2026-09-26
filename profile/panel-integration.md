# Интеграция с панелью

Как backend и node работают с fedarisha: API ноды, реакция на события пользователей, кеш ключей и сборка подписки.

Код:

- backend: [`src/modules/fedarisha-provisioning/`](https://github.com/Fedarisha/backend/tree/main/src/modules/fedarisha-provisioning), [`src/common/utils/apply-fedarisha-webhook-defaults.ts`](https://github.com/Fedarisha/backend/blob/main/src/common/utils/apply-fedarisha-webhook-defaults.ts), [`resolve-proxy-config.service.ts`](https://github.com/Fedarisha/backend/blob/main/src/modules/subscription-template/resolve-proxy/resolve-proxy-config.service.ts);
- node: [`src/modules/fedarisha-pak/`](https://github.com/Fedarisha/node/tree/main/src/modules/fedarisha-pak), ветка `fedarisha` в [`src/modules/handler/handler.service.ts`](https://github.com/Fedarisha/node/blob/main/src/modules/handler/handler.service.ts).

## Пользователи в Xray ноды

Используется стандартный механизм Remnawave. При сборке конфига для ноды backend добавляет в каждый fedarisha-инбаунд `settings.clients` вида `{"id": "<id>", "email": "<id>"}`, где `<id>` — числовой id пользователя. При изменениях без перезапуска Xray backend вызывает `/node/handler/add-user` и `remove-user` с пользователем `{"type": "fedarisha", "tag": "<тег>", "username": "<id>"}`.

## API ноды

Три эндпоинта с тем же mTLS и JWT, что и остальное API ноды. Все принимают JSON методом `POST` и отвечают HTTP 200. Результат передаётся в поле `response.isOk`. Если тело не проходит проверку схемы, нода возвращает 400.

Инбаунд нода ищет по `inboundTag` в конфиге своего Xray, который получила от панели при старте.

### `POST /node/fedarisha/provision-user`

Выпустить ключ пользователю. Таймаут на стороне backend — 20 с.

```json
{ "userUuid": "…", "inboundTag": "fed-eu", "prefix": "fed/42/" }
```

```json
{ "response": { "isOk": true, "accessKey": "…", "secretKey": "…", "error": null } }
```

При ошибке: `isOk: false`, ключи `null`, в `error` описание. Например, `fedarisha inbound fed-eu not found in xray config`: такой ответ бывает, если тега нет в конфиге ноды или у инбаунда не задан корректный `authType` либо обязательные поля провайдера.

### `POST /node/fedarisha/revoke-user`

Отозвать ключ. Таймаут 20 с.

```json
{ "userUuid": "…", "inboundTag": "fed-eu", "prefix": "fed/42/" }
```

```json
{ "response": { "isOk": true, "error": null } }
```

### `POST /node/fedarisha/probe-user`

Проверить, что ключ ещё действует (см. [storage-providers.md](storage-providers.md#проверка-ключа)). Таймаут 10 с.

```json
{ "userUuid": "…", "inboundTag": "fed-eu", "prefix": "fed/42/", "accessKey": "…", "secretKey": "…" }
```

| Ответ | Значение |
| --- | --- |
| `isOk: true, exists: true` | Ключ действует. |
| `isOk: true, exists: false` | S3 отклонил ключ (403/404). |
| `isOk: false` | Проверить не удалось: сбой сети, неожиданный ответ S3 или ошибка конфига. |

`userUuid` — числовой id пользователя (название поля историческое, UUID тоже принимается для совместимости со старыми панелями). Он входит в имя ключа у провайдера.

## События пользователей

Обработчики в `FedarishaProvisioningEvents` выполняются асинхронно. Их ошибки только пишутся в лог.

| Событие | Действие |
| --- | --- |
| `user.enabled`, `user.modified`, `user.traffic_reset` при статусе `ACTIVE` | Выпустить ключи для fedarisha-инбаундов из сквадов пользователя, для которых ключа ещё нет в кеше. |
| `user.disabled`, `user.limited`, `user.expired`, `user.deleted` | Отозвать все ключи пользователя и очистить кеш. |

На `user.created` обработчика нет. Новый пользователь получает ключ при первом запросе подписки.

Предварительный выпуск по событиям нужен потому, что клиенты обновляют подписку редко (обычно раз в 12 часов). Без него пользователь, которому, например, добавили трафик, ждал бы ключ до следующего обновления.

При отзыве для каждого ключа из кеша backend находит включённую ноду с тем же конфиг-профилем (`configProfileUuid` хранится вместе с ключом) и вызывает `revoke-user`. Кеш очищается в любом случае, даже если нода недоступна или вернула ошибку. В этом случае ключ остаётся у провайдера, и удалять его придётся вручную.

## Кеш ключей

Ключи хранятся в `user_meta.metadata.fedarisha`, по одному на тег инбаунда:

```json
{
  "fed-eu": {
    "accessKey": "…",
    "secretKey": "…",
    "prefix": "fed/42/",
    "configProfileUuid": "…",
    "issuedAt": "2026-09-26T12:00:00.000Z"
  }
}
```

Ключом записи служит тег, поэтому при переименовании тега кеш теряется, и ключ будет выпущен заново. `issuedAt` на логику не влияет.

## Как backend получает ключ (`ensureCredentials`)

Используется и при предварительном выпуске, и при сборке подписки.

1. Ожидаемый префикс: `<storage.prefix без завершающего "/">/<id>/`, или `<id>/`, если `prefix` пустой.
2. Если в кеше есть ключ с таким же префиксом, backend вызывает `probe-user`:
   - `exists: true` — использовать ключ из кеша;
   - `isOk: false` или сбой сети — тоже использовать ключ из кеша, чтобы пользователь не остался без подписки из-за временной недоступности ноды;
   - `exists: false` — выпустить новый ключ.
3. Если в кеше лежит ключ с другим префиксом (изменился `storage.prefix`), старый ключ сначала отзывается.
4. Вызывается `provision-user`. Новый ключ записывается в кеш.
5. Если выпуск не удался, возвращается `null`, и хост не попадает в подписку.

Нода для вызовов выбирается так: сначала включённая нода, которой назначен этот инбаунд в текущем профиле, иначе любая включённая нода с этим профилем.

Для каждого fedarisha-хоста это означает один вызов `probe-user` при каждом запросе подписки, то есть PUT, HEAD и DELETE к S3 от имени ключа пользователя.

## Подписка

Fedarisha-хосты попадают только в ответ на `/<shortUuid>/fedarisha-json`. Этот тип отдаёт тот же Xray JSON, что и `json`, но с fedarisha-хостами. Во всех остальных типах (`json`, `v2ray-json`, `singbox`, `mihomo`, `clash`, base64-ссылки) такие хосты пропускаются, чтобы не ломать клиенты без поддержки протокола. Subscription-page пропускает `fedarisha-json` в свой список допустимых типов и передаёт запрос в backend.

Чтобы инбаунд попал в подписку, для него нужен хост в панели. Адрес, порт и параметры TLS хоста для fedarisha не используются. Если у нескольких хостов один инбаунд, ключ выпускается один раз.

Outbound собирается из инбаунда и ключа пользователя:

```json
{
  "protocol": "fedarisha",
  "settings": {
    "storage": {
      "type": "s3",
      "bucket": "<из инбаунда>",
      "endpoint": "<из инбаунда>",
      "region": "<из инбаунда>",
      "prefix": "fed/42/",
      "sessionsDir": "<из инбаунда или sessions>",
      "accessKey": "<ключ пользователя>",
      "secretKey": "<ключ пользователя>"
    },
    "tuning": { … из инбаунда … }
  }
}
```

`authType`, `iam`, `webhook`, `clients` и мастер-ключи в outbound не попадают. `type` всегда `s3`. `streamSettings` не добавляются. `mux` и маппер хоста применяются так же, как для других протоколов.

## Значения webhook по умолчанию

Перед отправкой конфига на ноду backend заполняет недостающие поля `settings.webhook` у fedarisha-инбаундов. Это происходит, только если `storage.type` равен `s3`, `authType` — одно из `vkcloud-pak`, `selectel-iam`, `static`, а блок `webhook` есть в конфиге.

| Поле | Значение |
| --- | --- |
| `enabled` | `true` |
| `listen` | `":80"` |
| `publicUrl` | `http://<адрес ноды в панели>/webhook` |
| `autoSetup` | `true` |

Явно заданные поля не перезаписываются. Отсюда три варианта:

- блока `webhook` нет — webhook выключен;
- `"webhook": {}` — webhook включён со значениями из таблицы;
- `"webhook": {"enabled": false}` — webhook выключен.

## Диагностика

| Симптом | Где смотреть | Вероятная причина |
| --- | --- | --- |
| Хоста нет в подписке `fedarisha-json` | Логи backend: `Skipping fedarisha inbound …`, `no enabled node serves it` | Нода выключена или недоступна, либо ей не назначен инбаунд, либо выпуск ключа завершился ошибкой. |
| `fedarisha inbound … not found in xray config` | Логи node | Тега нет в конфиге Xray ноды, у инбаунда нет корректного `authType` или не заданы `bucket`, `endpoint`, мастер-ключи (для Selectel ещё блок `iam`). |
| Клиент получил конфиг, но не подключается (`server did not ACK within 60s`) | Логи Xray на ноде | Пользователя нет в `clients` (в логе `rejected: user "…" not allowed`), у мастер-ключа нет прав на LIST, `sessionsDir` отличается у клиента и ноды. |
| Нода долго замечает новые сессии | Логи Xray: `configuring S3 webhook`, `WARNING webhook setup failed` | Webhook не настроен или `publicUrl` недоступен провайдеру. Без webhook сессия обнаруживается в пределах 500 мс. |
| `WARNING lifecycle setup failed` | Логи Xray | У мастер-ключа нет прав на lifecycle, или провайдер не поддерживает это API. Брошенные сессии придётся удалять вручную. |
| `Selectel: bucket policy exceeds the 20 KB limit` | Логи node | Слишком много пользователей на бакет. Разнесите их по нескольким инбаундам с разными бакетами. |
| `Selectel: storage.masterServiceUserId …`, `storage.accessKey does not belong to …` | Логи node | `masterServiceUserId` не задан, задан неверно или `accessKey` принадлежит другому пользователю. |
