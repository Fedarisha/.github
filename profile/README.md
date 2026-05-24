# Fedarisha

Форк [Remnawave](https://github.com/remnawave) с собственным транспортом `fedarisha`: prefix-scoped S3-доставка PAK-токенов вместо классических Trojan/VLESS-секретов.

Все образы публикуются в Docker Hub и GHCR — собирать ничего не нужно.

| Сервис | Образ |
| --- | --- |
| xray-core | `voltara13/xray-core:latest` · `ghcr.io/fedarisha/xray-core:latest` |
| node | `voltara13/node:latest` · `ghcr.io/fedarisha/node:latest` |
| backend (+ frontend) | `voltara13/backend:latest` · `ghcr.io/fedarisha/backend:latest` |
| subscription-page | `voltara13/subscription-page:latest` · `ghcr.io/fedarisha/subscription-page:latest` |

Теги — `vX.Y.Z-fed.N` (`X.Y.Z` — апстрим, `-fed.N` — счётчик форка).

## Документация

**Для оператора (deploy & config):**
- **[quickstart.md](quickstart.md)** — деплой панели, sub-page, ноды, reverse-proxy и включение fedarisha-инбаунда.
- **[inbound-config.md](inbound-config.md)** — полная схема fedarisha-инбаунда: storage / tuning / webhook / clients.
- **[storage-providers.md](storage-providers.md)** — конфигурация S3-провайдеров (VK Cloud PAK, Selectel IAM, Static).

**Для понимания того, как работает форк:**
- **[architecture.md](architecture.md)** — карта компонентов, жизненный цикл пользователя, уровни изоляции, где хранится state.
- **[protocol.md](protocol.md)** — wire-протокол fedarisha-транспорта: раскладка в бакете, handshake, encryption, tuning, webhook.
- **[node-api.md](node-api.md)** — REST-контракт `/node/fedarisha/{provision,revoke,probe}-user`, что валидируется, как авторизуется.
- **[subscription-flow.md](subscription-flow.md)** — event-driven provisioning, ensureCredentials, рендер outbound для подписки, failure modes.

**Для разработчика форка:**
- **[build-from-source.md](build-from-source.md)** — что клонировать, цепочка сборки, релизный workflow.

## Что внутри форка

- Свой transport `fedarisha` в [Xray-core-fedarisha](https://github.com/Fedarisha/Xray-core-fedarisha) — выдаёт пользователю PAK-токен и ссылку на S3-объект вместо встроенного секрета.
- [node](https://github.com/Fedarisha/node) `src/modules/fedarisha-pak/*` — провижн/ревок S3-доступа через VK Cloud PAK или Selectel IAM, REST `/node/fedarisha/{provision,revoke,probe}-user`.
- [backend](https://github.com/Fedarisha/backend) `src/modules/fedarisha-provisioning/*` — оркестратор: на createUser/updateUser/deleteUser зовёт ноды, кеширует выданные ключи, разруливает миграции между inbound-ами.
- [subscription-page](https://github.com/Fedarisha/subscription-page) — отдаёт client-type `fedarisha-json` на `/{shortUuid}/fedarisha-json`, который зашивается в форки клиентов ([v2rayN](https://github.com/Fedarisha/v2rayN), [v2rayNG](https://github.com/Fedarisha/v2rayNG)).
