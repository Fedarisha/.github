# Установка

Панель и ноды ставятся так же, как в [Remnawave](https://docs.rw), но с образами Fedarisha. Этот документ описывает только отличия и настройку fedarisha-инбаунда. Всё остальное (compose-файлы, `.env`, reverse proxy, регистрация нод) делается по официальной документации Remnawave.

## 1. Образы

Замените образы в compose-файлах из инструкции Remnawave:

| Сервис | Было | Стало |
| --- | --- | --- |
| Панель | `remnawave/backend` | `ghcr.io/fedarisha/backend` |
| Нода | `remnawave/node` | `ghcr.io/fedarisha/node` |
| Страница подписки | `remnawave/subscription-page` | `ghcr.io/fedarisha/subscription-page` |

Указывайте теги одного релиза Fedarisha (например, `3.4.4-1.0.1fed` для backend и `3.4.1-1.0.1fed` для node), а не `latest`. Frontend уже встроен в образ backend.

## 2. Бакет

Для fedarisha нужен отдельный бакет. Инбаунд ставит в нём правило удаления объектов старше суток под своим `prefix`. Провайдер `selectel-iam` к тому же целиком перезаписывает bucket policy. Хранить в этом бакете что-то ещё не стоит.

Мастер-ключу нужны права на чтение, запись, удаление и LIST объектов, HeadBucket и настройку lifecycle. Для `vkcloud-pak` также нужны права на выпуск PAK, а для webhook — на настройку уведомлений бакета.

## 3. Нода

Используйте стандартный compose ноды Remnawave (`network_mode: host`, `NODE_PORT`, `SECRET_KEY`) с образом `ghcr.io/fedarisha/node`. Xray с поддержкой fedarisha уже есть в образе.

Если будете использовать webhook:

- Порт из `webhook.listen` (по умолчанию `:80`) должен быть свободен на хосте и доступен из сети провайдера S3.
- По умолчанию `publicUrl` собирается из адреса ноды в панели: `http://<адрес>/webhook`. Этот адрес должен быть доступен провайдеру уже при старте Xray, иначе регистрация webhook не пройдёт (нода продолжит работать через опрос).
- Если нужен TLS, задайте `tlsCert` и `tlsKey` (пути внутри контейнера) или поставьте перед нодой reverse proxy и укажите свой `publicUrl`.

## 4. Инбаунд в конфиг-профиле

Добавьте в конфиг-профиль панели инбаунд. Пример для VK Cloud:

```json
{
  "tag": "fed-eu",
  "protocol": "fedarisha",
  "settings": {
    "storage": {
      "type": "s3",
      "authType": "vkcloud-pak",
      "bucket": "my-fedarisha-bucket",
      "endpoint": "https://hb.ru-msk.vkcloud-storage.ru",
      "region": "ru-msk",
      "prefix": "fed",
      "accessKey": "<мастер-ключ>",
      "secretKey": "<мастер-секрет>"
    },
    "webhook": {}
  }
}
```

- `authType` выбирает, как нода выдаёт ключи пользователям: `vkcloud-pak`, `selectel-iam` или `static`. Для Selectel нужны ещё `masterServiceUserId` и блок `iam`, см. [storage-providers.md](storage-providers.md).
- `"webhook": {}` включает webhook со значениями по умолчанию. Уберите блок, если webhook не нужен.
- `clients` не указывайте, их заполняет панель.
- `tuning` необязателен, значения по умолчанию подобраны под скорость. См. [xray-config.md](xray-config.md#tuning).

Мастер-ключи хранятся в конфиг-профиле открытым текстом и видны всем администраторам панели.

## 5. Назначение

1. Включите инбаунд на нужной ноде.
2. Добавьте инбаунд во внутренний сквад и добавьте в сквад пользователей.
3. Создайте хост для этого инбаунда. Без хоста инбаунд не попадёт в подписку. Адрес и порт хоста не используются, укажите любые допустимые значения.

После перезапуска Xray на ноде в логах должны появиться строки:

```
[fedarisha] inbound "fed-eu": configuring S3 bucket lifecycle (prefix: fed/, expire: 1d)
[fedarisha] inbound "fed-eu": configuring S3 webhook http://<адрес>/webhook (prefix: fed/)
[fedarisha] inbound "fed-eu": webhook enabled on :80/webhook
```

## 6. Подписка

Адрес подписки для клиентов с поддержкой fedarisha:

```
https://<домен страницы подписки>/<shortUuid>/fedarisha-json
```

Это Xray JSON, в котором fedarisha-хосты идут вместе с остальными. По обычной ссылке подписки fedarisha-хостов нет.

Ссылку нужно добавить в клиент с ядром Xray-core-fedarisha: [v2rayN](https://github.com/voltara13/v2rayN) или [v2rayNG](https://github.com/voltara13/v2rayNG). При первом запросе подписки панель выпустит пользователю ключ. Если подписка пришла без fedarisha-хоста, смотрите раздел [Диагностика](panel-integration.md#диагностика).
