# Fedarisha

Здесь живут два разных проекта, между которыми важно не путаться:

- **[Xray-core-fedarisha](https://github.com/Fedarisha/Xray-core-fedarisha)** — самостоятельный форк xray-core, в который добавлен transport `fedarisha` (S3-бакет как канал связи). Два xray-узла + общий бакет = рабочий VPN. Никакая панель для этого не нужна — это обычный xray-config с одним нестандартным протоколом.
- **[node](https://github.com/Fedarisha/node) + [backend](https://github.com/Fedarisha/backend) + [subscription-page](https://github.com/Fedarisha/subscription-page)** — форки [Remnawave](https://github.com/remnawave), которые поднимают над этим транспортом панель управления: автоматически выдают per-user PAK-ключи, рулят lifecycle юзеров, рендерят подписки в форки клиентов.

То есть: транспорт сам по себе ничего про «пользователей панели» не знает — он просто умеет писать и читать зашифрованные кадры в общем S3-префиксе. Remnawave-стек добавляет multi-tenancy, ротацию ключей и подписки поверх.

Зачем такое нужно: DPI видит обычные PUT/GET к публичному облаку (`hb.ru-msk.vkcloud-storage.ru` и т.п.), а не VPN-трафик. Чтобы заблокировать форк, провайдеру нужно отрезать целое облако. Цена — латентность 50–250 мс и стоимость S3-запросов.

## Что брать

Все образы в Docker Hub — собирать ничего не нужно.

| Сервис | Образ | Слой |
| --- | --- | --- |
| xray-core | `voltara13/xray-core:latest` | транспорт |
| node | `voltara13/node:latest` | Remnawave |
| backend (+ frontend) | `voltara13/backend:latest` | Remnawave |
| subscription-page | `voltara13/subscription-page:latest` | Remnawave |

Теги — `vX.Y.Z-fed.N` (`X.Y.Z` — апстрим, `-fed.N` — счётчик форка).

Клиенты:
- **[v2rayN](https://github.com/Fedarisha/v2rayN)** / **[v2rayNG](https://github.com/Fedarisha/v2rayNG)** — форки UI-клиентов со встроенным xray-core-fedarisha и пониманием подписочного `fedarisha-json`.
- Чистый xray-core-fedarisha можно использовать и без UI — обычный `xray run -c config.json`.

## Документация

### Если нужен только транспорт (без панели)

- **[inbound-config.md](inbound-config.md)** — полная схема fedarisha-инбаунда в xray-config.
- **[storage-providers.md](storage-providers.md)** — как сконфигурить `storage` для разных S3 (раздел про `type: "s3"` / `static` — это то, что нужно standalone-сценарию).
- **[protocol.md](protocol.md)** — wire-протокол: что лежит в бакете, как устроен handshake, как фреймится трафик. Это «спецификация», не зависит от Remnawave.

### Если разворачиваете полный Remnawave-стек

- **[quickstart.md](quickstart.md)** — собрать compose-стек панели, sub-page, ноды и включить fedarisha-инбаунд.
- **[architecture.md](architecture.md)** — как уложены два слоя, что делает каждый, и жизнь одного пользователя от создания в панели до первого кадра в бакете.
- **[storage-providers.md](storage-providers.md)** — какой PAK-провайдер выбрать на ноде (VK Cloud PAK / Selectel IAM / static).
- **[node-api.md](node-api.md)** — REST-контракт `/node/fedarisha/{provision,revoke,probe}-user` (это контракт **panel ↔ node**, не клиент ↔ node).
- **[subscription-flow.md](subscription-flow.md)** — event-driven provisioning в бэкенде, `ensureCredentials`, рендер outbound, failure modes.

### Если правите форк

- **[build-from-source.md](build-from-source.md)** — что клонировать, цепочка сборки, релизный workflow.
