# 03. Синхронизация Obsidian через свой сервер

Предыдущая: [02-docker и traefik](02-docker%20%D0%B8%20traefik.md). Следующая: [04-мост в файлы (livesync-bridge)](04-%D0%BC%D0%BE%D1%81%D1%82%20%D0%B2%20%D1%84%D0%B0%D0%B9%D0%BB%D1%8B%20%28livesync-bridge%29.md).

## Выбор технологии

| Критерий | Syncthing | CouchDB + LiveSync |
|---|---|---|
| Файлы на сервере сразу | да, в этом суть | нет, нужен мост |
| iOS | нет клиента | да, плагин внутри Obsidian |
| Компонентов на сервере | 1 | 2 (CouchDB и мост) |
| RAM | около 50 MB | около 300-500 MB суммарно |

Если нужна работа на iPhone, используйте CouchDB + LiveSync. Если iOS не нужен, подойдет Syncthing: тогда пропустите эту и следующую главы.

## Сервер: CouchDB за Traefik

Создайте каталоги и файл с секретами:

```bash
sudo mkdir -p /opt/second-brain/{couchdb/data,couchdb/etc/local.d}
echo "COUCHDB_USER=sync_user
COUCHDB_PASSWORD=$(openssl rand -hex 24)" | sudo tee /opt/second-brain/.env
sudo chmod 600 /opt/second-brain/.env
```

Создайте /opt/second-brain/compose.yml. Метки Traefik заданы по шаблону из [02-docker и traefik](02-docker%20%D0%B8%20traefik.md):

```yaml
services:
  couchdb:
    image: couchdb:3
    container_name: second-brain-couchdb
    restart: unless-stopped
    env_file: [.env]
    environment:
      - COUCHDB_USER=${COUCHDB_USER}
      - COUCHDB_PASSWORD=${COUCHDB_PASSWORD}
    volumes:
      - ./couchdb/data:/opt/couchdb/data
      - ./couchdb/etc/local.d:/opt/couchdb/etc/local.d
    networks: [traefik_network, second-brain-internal]
    labels:
      - traefik.enable=true
      - traefik.http.routers.sync.rule=Host(`sync.example.com`)
      - traefik.http.routers.sync.tls=true
      - traefik.http.routers.sync.tls.certresolver=mytlschallenge
      - traefik.http.routers.sync.entrypoints=web,websecure
      - traefik.http.routers.sync.middlewares=sync-rl
      - traefik.http.services.sync.loadbalancer.server.port=5984
      - traefik.http.middlewares.sync-rl.ratelimit.average=50
      - traefik.http.middlewares.sync-rl.ratelimit.burst=100
      - traefik.docker.network=traefik_network

networks:
  traefik_network: {external: true}
  second-brain-internal: {name: second-brain-internal}
```

Создайте файл настроек couchdb/etc/local.d/10-livesync.ini (в каталоге /opt/second-brain):

```ini
[couchdb]
single_node = true
max_document_size = 50000000

[chttpd]
require_valid_user = true
max_http_request_size = 4294967296
enable_cors = true

[chttpd_auth]
require_valid_user = true

[cors]
credentials = true
origins = app://obsidian.md,capacitor://localhost,http://localhost,ionic://localhost
headers = accept, authorization, content-type, origin, referer
methods = GET, PUT, POST, HEAD, DELETE, OPTIONS
max_age = 3600
```

Строка origins - список адресов, которым разрешено обращаться к базе из браузерного движка: app://obsidian.md для компьютера, capacitor://localhost и ionic://localhost для iOS и Android. Сверьте ее посимвольно. Опечатка в любом слове дает на устройстве ошибку «Failed to fetch ... check CORS» при работающем сервере. Фактические значения проверяются командой:

```bash
sudo docker exec second-brain-couchdb curl -s -u 'user:pass' http://localhost:5984/_node/_local/_config/cors
```

Запуск: создайте DNS-запись, затем проверьте файл и запустите:

```bash
sudo docker compose -f /opt/second-brain/compose.yml config --quiet && echo OK
sudo docker compose -f /opt/second-brain/compose.yml up -d
sudo docker exec second-brain-couchdb curl -s -u 'user:pass' http://localhost:5984/_up
```

Признак успеха: ответ {"status":"ok"}.

Затем откройте в браузере https://sync.example.com/_utils, выберите Create Database и задайте имя базы, например obsidiandb (тип non-partitioned). Одна база соответствует одному хранилищу. Для второго хранилища создайте вторую базу, не смешивайте.

Административные команды выполняйте изнутри контейнера (docker exec), не через публичный домен: так на результат не влияют прокси-переменные сессии и редиректы.

## Клиент 1: компьютер

1. Установите плагин Self-hosted LiveSync (Settings, Community plugins).
2. Отключите конкурентов. В официальном Obsidian Sync выберите Disconnect. Хранилище не должно лежать в папке iCloud, OneDrive или Dropbox. Два синхронизатора на одних файлах дают конфликты и дубли.
3. Настройте вручную: Remote URI https://sync.example.com, логин, пароль и имя базы. Нажмите Test, должно быть зеленое подтверждение.
4. Выключите E2EE. Серверу (мосту и агентам) нужен читаемый текст. Защита обеспечивается TLS, аутентификацией и файрволом.
5. Выполните первичную отправку: Rebuild/Overwrite remote (локальные заметки в пустую базу). Операции «Schedule and Restart» выполняются после полного перезапуска Obsidian (Ctrl+Q), а не после сворачивания окна.
6. Проверьте: счетчик документов в /_utils вырос. Документов больше, чем заметок, потому что заметки хранятся кусочками.

## Клиент 2 и следующие: iPhone и другие устройства

1. На настроенном компьютере выберите Copy settings as a new Setup URI, задайте парольную фразу и перешлите строку себе.
2. На iPhone создайте новое пустое хранилище (не поверх существующего). Имя хранилища локальное и на синхронизацию не влияет. Установите тот же плагин, выберите Enter Setup URI и введите фразу.
3. Выберите направление инициализации Fetch (сервер на устройство). Выбор Rebuild remote затрет базу пустым хранилищем телефона.
4. На iOS синхронизация идет, пока Obsidian открыт. После записи заметки подождите пару секунд. При ошибке CORS на iPhone при работающем компьютере проверьте строку origins (capacitor и ionic), затем полностью закройте приложение свайпом и откройте снова.

## Двусторонний финальный тест

Создайте заметку на телефоне: через несколько секунд она должна появиться на компьютере. Измените заметку на компьютере: правка должна появиться на телефоне. Этап закрыт только после этого.

## Обслуживание

- Смена пароля CouchDB: измените значение в .env, удалите файл docker.ini из couchdb/etc/local.d (в нем хеш старого пароля администратора), пересоздайте контейнер командой `sudo docker compose -f /opt/second-brain/compose.yml up -d --force-recreate`, введите новый пароль в плагине на всех устройствах. Данные базы остаются.
- Предупреждение «database is locked» после Rebuild нормально: выполните Fetch на других устройствах, затем снимите блокировку (unlock).
- Периодически открывайте /_utils: там видно размер базы и состояние.

## Чек-лист выхода

- [ ] _up отвечает ok, аутентификация обязательна (анонимный запрос дает 401)
- [ ] CORS-origins сверены посимвольно
- [ ] Двусторонний тест пройден на всех устройствах
- [ ] E2EE выключен, пароль хранится только в .env
- [ ] Конкурирующие синхронизаторы отключены на всех устройствах
