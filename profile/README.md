# Fedarisha

Fedarisha — это VPN-транспорт, в котором клиент и сервер не соединяются друг с другом напрямую. Они обмениваются зашифрованными файлами через общий S3-бакет. Для сетевого наблюдателя клиент выглядит как обычное приложение, которое делает PUT/GET/LIST к публичному объектному хранилищу (VK Cloud, Selectel и т.п.). Адреса VPN-сервера в трафике клиента нет.

Транспорт встроен в форк Xray-core как протокол `fedarisha`. Поверх него есть форк панели Remnawave: он выдаёт каждому пользователю собственный S3-ключ с доступом только к его префиксу и собирает подписки.

## Как это работает

```
 клиент (xray)                    S3-бакет                       нода (xray)
 outbound fedarisha  ── PUT c_* ──►  <prefix>/<user>/sessions/<id>/  ◄── LIST/GET ──  inbound fedarisha ──► интернет
                     ◄── GET s_* ──                                  ── PUT s_* ──
```

1. Клиент создаёт в бакете папку сессии и кладёт в неё `c_hello` со своим X25519-ключом.
2. Нода находит новую сессию (через LIST или по webhook от S3) и отвечает файлом `s_ack` со своим ключом. Обе стороны выводят общий ключ AES-256-GCM.
3. Дальше каждая сторона пишет пронумерованные зашифрованные файлы (`c_00000000`, `s_00000000`, …) и читает файлы другой стороны. Прочитанные файлы сразу удаляются.
4. Внутри этого байтового потока работает yamux: все TCP- и UDP-соединения клиента мультиплексируются в одну S3-сессию.

Подробно: [protocol.md](protocol.md).

## Из чего состоит

| Репозиторий | Что это | Что добавлено в форке |
| --- | --- | --- |
| [Xray-core-fedarisha](https://github.com/Fedarisha/Xray-core-fedarisha) | Форк [XTLS/Xray-core](https://github.com/XTLS/Xray-core) | Протокол `fedarisha` (inbound и outbound), `proxy/fedarisha/` |
| [node](https://github.com/Fedarisha/node) | Форк [remnawave/node](https://github.com/remnawave/node) | Xray с fedarisha, API выдачи S3-ключей `/node/fedarisha/*` |
| [backend](https://github.com/Fedarisha/backend) | Форк [remnawave/backend](https://github.com/remnawave/backend) | Выдача и отзыв ключей по событиям пользователей, рендер fedarisha-outbound в подписке |
| [frontend](https://github.com/Fedarisha/frontend) | Форк [remnawave/frontend](https://github.com/remnawave/frontend) | Редактор конфигов принимает инбаунды `fedarisha` |
| [subscription-page](https://github.com/Fedarisha/subscription-page) | Форк [remnawave/subscription-page](https://github.com/remnawave/subscription-page) | Тип клиента `fedarisha-json` |

Клиенты:

- [voltara13/v2rayN](https://github.com/voltara13/v2rayN): форк v2rayN для Windows, использует ядро из [Fedarisha/xray-builds](https://github.com/Fedarisha/xray-builds);
- [voltara13/v2rayNG](https://github.com/voltara13/v2rayNG): форк v2rayNG для Android со встроенным Xray-core-fedarisha;
- [Fedarisha/client](https://github.com/Fedarisha/client): клиент на Flutter для Android и iOS на базе [libXray-fedarisha](https://github.com/Fedarisha/libXray-fedarisha), находится в разработке.

Транспорт работает и без панели. Достаточно двух экземпляров Xray-core-fedarisha и бакета, см. [xray-config.md](xray-config.md#минимальный-пример-без-панели).

## Образы

| Сервис | Образ |
| --- | --- |
| backend (со встроенным frontend) | `ghcr.io/fedarisha/backend` |
| node | `ghcr.io/fedarisha/node` |
| subscription-page | `ghcr.io/fedarisha/subscription-page` |

Теги имеют вид `<версия апстрима>-<версия Fedarisha>fed`, например `3.4.4-1.0.1fed` у backend и `v26.9.9-1.0.1fed` у Xray. Версия Fedarisha общая для всех репозиториев одного релиза. Образы также зеркалируются в Docker Hub под `voltara13/*`. Бинарники Xray лежат в релизах [Xray-core-fedarisha](https://github.com/Fedarisha/Xray-core-fedarisha/releases).

## Документация

- [architecture.md](architecture.md): компоненты, путь пользователя от создания до первого пакета, где хранится состояние, модель безопасности.
- [protocol.md](protocol.md): что лежит в бакете, рукопожатие, формат файлов, опрос, webhook, очистка.
- [xray-config.md](xray-config.md): все поля inbound и outbound `fedarisha`, пример без панели.
- [storage-providers.md](storage-providers.md): как нода выдаёт ключи на VK Cloud, Selectel и в режиме `static`.
- [quickstart.md](quickstart.md): установка панели и ноды, добавление fedarisha-инбаунда.
- [panel-integration.md](panel-integration.md): API ноды, события backend, кеш ключей, рендер подписки, диагностика.
- [build-from-source.md](build-from-source.md): сборка и выпуск релизов.

## Ограничения

- Задержка заметно выше, чем у обычного VPN: каждый переход данных — это PUT, затем LIST и GET на другой стороне. Интерактивные приложения работают медленнее.
- Каждая сессия постоянно опрашивает бакет. Платные S3-запросы (LIST, PUT, GET, DELETE) составляют основную часть стоимости эксплуатации.
- Выдача ключей по пользователям реализована для VK Cloud (`vkcloud-pak`) и Selectel (`selectel-iam`). Для остальных S3 есть только режим `static`: один общий ключ на всех, без изоляции.
- Автонастройка webhook использует расширение API VK Cloud. У других провайдеров нода работает через опрос.
