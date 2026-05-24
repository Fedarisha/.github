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

- **[quickstart.md](quickstart.md)** — деплой панели, sub-page, ноды, reverse-proxy и включение fedarisha-инбаунда.
- **[storage-providers.md](storage-providers.md)** — конфигурация S3-провайдеров (VK Cloud PAK, Selectel IAM).
- **[build-from-source.md](build-from-source.md)** — что клонировать, цепочка сборки, релизный workflow.

## Что внутри форка

- Свой transport `fedarisha` в [Xray-core-fedarisha](https://github.com/Fedarisha/Xray-core-fedarisha) — выдаёт пользователю PAK-токен и ссылку на S3-объект вместо встроенного секрета.
- [node](https://github.com/Fedarisha/node) `src/modules/fedarisha-pak/*` — провижн/ревок S3-доступа через VK Cloud PAK или Selectel IAM, REST `/node/fedarisha/{provision,revoke,probe}-user`.
- [backend](https://github.com/Fedarisha/backend) `src/modules/fedarisha-provisioning/*` — оркестратор: на createUser/updateUser/deleteUser зовёт ноды, кеширует выданные ключи, разруливает миграции между inbound-ами.
- [subscription-page](https://github.com/Fedarisha/subscription-page) — отдаёт client-type `fedarisha-json` на `/{shortUuid}/fedarisha-json`, который зашивается в форки клиентов ([v2rayN](https://github.com/Fedarisha/v2rayN), [v2rayNG](https://github.com/Fedarisha/v2rayNG)).
