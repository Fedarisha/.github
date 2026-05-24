# Fedarisha

Форк [Remnawave](https://github.com/remnawave), в котором клиент общается с нодой **через файлы в S3-бакете**, а не через TCP. DPI видит обычные PUT/GET к публичному облаку — не VPN; чтобы заблокировать форк, провайдеру нужно отрезать целое облако (VK Cloud, Selectel, что вы поставите).

Цена — латентность 50–250 мс и стоимость S3-запросов. Зато блокировка по IP ноды больше не работает.

## Что брать

Все образы публикуются в Docker Hub — собирать ничего не нужно.

| Сервис | Образ |
| --- | --- |
| xray-core | `voltara13/xray-core:latest` |
| node | `voltara13/node:latest` |
| backend (+ frontend) | `voltara13/backend:latest` |
| subscription-page | `voltara13/subscription-page:latest` |

Теги — `vX.Y.Z-fed.N` (`X.Y.Z` — апстрим, `-fed.N` — счётчик форка).

## Что внутри

- **[Xray-core-fedarisha](https://github.com/Fedarisha/Xray-core-fedarisha)** — сам транспорт `fedarisha` в xray-core: знает, как клиент и инбаунд общаются через файлы в S3-папке (handshake, шифрование, фреймы). PAK-ключи не выдаёт — берёт готовые из `settings.storage`.
- **[node](https://github.com/Fedarisha/node)** — `src/modules/fedarisha-pak/*`, провижн/ревок S3-доступа через VK Cloud PAK, Selectel IAM или static. REST `/node/fedarisha/{provision,revoke,probe}-user` — **именно здесь выпускаются PAK-ключи**.
- **[backend](https://github.com/Fedarisha/backend)** — `src/modules/fedarisha-provisioning/*`, оркестратор: на createUser/updateUser/deleteUser зовёт ноды, кеширует выданные ключи, рулит миграциями между inbound-ами.
- **[subscription-page](https://github.com/Fedarisha/subscription-page)** — отдаёт client-type `fedarisha-json` на `/{shortUuid}/fedarisha-json`, который зашивается в форки клиентов ([v2rayN](https://github.com/Fedarisha/v2rayN), [v2rayNG](https://github.com/Fedarisha/v2rayNG)).

## Документация

**Если разворачиваете:**

- **[quickstart.md](quickstart.md)** — собрать compose-стек панели, sub-page, ноды и включить fedarisha-инбаунд.
- **[inbound-config.md](inbound-config.md)** — полная схема fedarisha-инбаунда: `storage`/`tuning`/`webhook`/`clients`.
- **[storage-providers.md](storage-providers.md)** — какой S3-провайдер выбрать и как его конфигурить (VK Cloud PAK, Selectel IAM, Static).

**Если разбираетесь, как форк устроен:**

- **[architecture.md](architecture.md)** — карта компонентов и жизнь одного пользователя от `USER.ENABLED` до первого кадра в бакете.
- **[protocol.md](protocol.md)** — wire-протокол: раскладка объектов, handshake, шифрование, tuning, webhook.
- **[node-api.md](node-api.md)** — REST-контракт `/node/fedarisha/{provision,revoke,probe}-user`.
- **[subscription-flow.md](subscription-flow.md)** — event-driven provisioning, `ensureCredentials`, рендер outbound, failure modes.

**Если правите форк:**

- **[build-from-source.md](build-from-source.md)** — что клонировать, цепочка сборки, релизный workflow.
