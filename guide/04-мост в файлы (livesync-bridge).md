# 04. Мост в файлы: livesync-bridge

Предыдущая: [03-синхронизация obsidian](03-%D1%81%D0%B8%D0%BD%D1%85%D1%80%D0%BE%D0%BD%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F%20obsidian.md). Следующая: [05-claude code на сервере](05-claude%20code%20%D0%BD%D0%B0%20%D1%81%D0%B5%D1%80%D0%B2%D0%B5%D1%80%D0%B5.md).

## Зачем он нужен

CouchDB хранит заметки кусочками документов. На диске сервера файлов нет, поэтому агентам работать не с чем. Мост livesync-bridge (автор LiveSync, vrtmrz) подключается к базе как еще один клиент и превращает содержимое в обычные .md-файлы. Работает в обе стороны: файл, созданный на сервере агентом или скриптом, попадает в базу и появляется на всех устройствах. Именно мост делает возможным управление агентами через заметки.

Установка идет слоями, у каждого слоя свой признак успеха.

## Установка

Перед началом убедитесь, что каталог хранилища /opt/second-brain/vault существует и принадлежит brain:secondbrain (он создан в главе [01-подготовка vps и безопасность](01-%D0%BF%D0%BE%D0%B4%D0%B3%D0%BE%D1%82%D0%BE%D0%B2%D0%BA%D0%B0%20vps%20%D0%B8%20%D0%B1%D0%B5%D0%B7%D0%BE%D0%BF%D0%B0%D1%81%D0%BD%D0%BE%D1%81%D1%82%D1%8C.md)).

### Слой 1: код и подмодуль

```bash
sudo git clone https://github.com/vrtmrz/livesync-bridge /opt/second-brain/bridge-app
sudo chown -R brain:secondbrain /opt/second-brain/bridge-app
sudo -u brain git -C /opt/second-brain/bridge-app submodule update --init --recursive
```

Каталог lib - это git-подмодуль (общая библиотека livesync-commonlib). Обычный clone оставляет его пустым, и при запуске появляется ошибка «Module not found .../logger.ts». Поэтому нужна последняя команда. Выполняйте ее от владельца каталога (sudo -u brain): иначе git откажет с сообщением «dubious ownership».

Признак успеха: каталог bridge-app/lib не пуст.

### Слой 2: npm-зависимости и кэш Deno

Мост работает на Deno, но зависит от npm-пакетов. Кэш Deno внутри образа принадлежит другому пользователю, поэтому при запуске от своего пользователя будет ошибка Permission denied. Обе задачи решает одноразовый контейнер с внешним кэшем:

```bash
sudo mkdir -p /opt/second-brain/deno-cache
sudo chown brain:secondbrain /opt/second-brain/deno-cache
sudo docker run --rm -v /opt/second-brain/bridge-app:/app -v /opt/second-brain/deno-cache:/deno-dir -e DENO_DIR=/deno-dir -w /app --user 1500:1600 denoland/deno:latest deno install --allow-scripts
```

Признак успеха: сообщение «Installed N packages». Предупреждения про устаревшие пакеты можно игнорировать.

### Слой 3: конфигурация

Создайте файл /opt/second-brain/bridge-app/dat/config.json. Подставьте имя базы, созданной в главе 03, и пароль из /opt/second-brain/.env:

```json
{
  "peers": [
    {
      "type": "couchdb",
      "name": "server-db",
      "url": "http://second-brain-couchdb:5984",
      "database": "obsidiandb",
      "username": "sync_user",
      "password": "ПАРОЛЬ",
      "customChunkSize": 100,
      "minimumChunkSize": 20,
      "passphrase": "",
      "obfuscatePassphrase": "",
      "baseDir": ""
    },
    {
      "type": "storage",
      "name": "server-files",
      "baseDir": "/vault/",
      "processor": {}
    }
  ]
}
```

- В поле url стоит внутреннее имя контейнера CouchDB: запрос идет мимо Traefik и интернета. Контейнеры должны быть в одной docker-сети.
- Пустое поле passphrase означает выключенный E2EE (решение из главы 03).
- Права на файл: 600, внутри пароль.

```bash
sudo chown brain:secondbrain /opt/second-brain/bridge-app/dat/config.json
sudo chmod 600 /opt/second-brain/bridge-app/dat/config.json
```

Проверьте JSON до запуска. Формат не прощает лишних запятых и точек с запятой:

```bash
sudo python3 -m json.tool /opt/second-brain/bridge-app/dat/config.json > /dev/null && echo OK
```

Признак успеха: слово OK.

### Слой 4: сервис в compose

Добавьте в файл /opt/second-brain/compose.yml, в раздел services:

```yaml
  livesync-bridge:
    image: denoland/deno:latest
    container_name: second-brain-bridge
    restart: unless-stopped
    working_dir: /app
    command: ["run", "-A", "main.ts"]
    user: "1500:1600"
    environment:
      - DENO_DIR=/deno-dir
    volumes:
      - ./bridge-app:/app
      - ./vault:/vault
      - ./deno-cache:/deno-dir
    networks: [second-brain-internal]
    depends_on: [couchdb]
```

Параметр user "1500:1600" делает владельцем создаваемых файлов brain. Сеть только внутренняя: наружу мосту выходить незачем.

## Запуск и проверка

```bash
sudo docker compose -f /opt/second-brain/compose.yml up -d
sudo docker logs second-brain-bridge --tail 50
```

Признак успеха в логе: скачивание модулей, затем «LiveSync Bridge is now starting», затем волна строк вида `[server-files] <-- ... saved`. Это материализация вашей базы в файлы. Проверьте:

```bash
ls -la /opt/second-brain/vault/
```

В каталоге должны лежать ваши папки и заметки.

## Двусторонний тест (обязательный)

1. Создайте заметку на любом устройстве. Через несколько секунд она должна появиться в `ls` на сервере.
2. Создайте файл на сервере:

```bash
sudo -u brain bash -c 'echo "# тест" > /opt/second-brain/vault/test-server.md'
```

Заметка test-server должна появиться на устройствах.

Если работают оба направления, основа системы готова.

## Эксплуатация

- Лог моста показывает все перемещения файлов. Чтобы узнать, кто и когда трогал файл: `sudo docker logs second-brain-bridge --since 1h | grep имя`.
- Разовые ошибки «delete failed» при первичной синхронизации ожидаемы: удаление пришло раньше, чем файл появился на диске.
- Мост может записывать один файл несколько раз (по кусочкам). Программы, которые реагируют на изменения в каталоге, должны это учитывать (глава [06-агенты и события](06-%D0%B0%D0%B3%D0%B5%D0%BD%D1%82%D1%8B%20%D0%B8%20%D1%81%D0%BE%D0%B1%D1%8B%D1%82%D0%B8%D1%8F.md)).
- Повторяющийся PermissionDenied в логе означает чужого владельца файла, а не сбой моста. Диагностика: `sudo docker logs second-brain-bridge --tail 200 | grep -i permissiondenied` показывает пути, для которых запись не проходит. Сравните их с результатом `sudo find /opt/second-brain/vault -not -user brain`. Лечение: `sudo chown -R brain:secondbrain` на затронутый каталог. Перезапуск с `--reset` не нужен: мост сам досинхронизирует после исправления прав. Чтобы не создавать такие ошибки, работайте с файлами хранилища на сервере только от brain (`sudo -u brain ...`). Разбор: [урок - права и владельцы в multi-user](../lessons/%D1%83%D1%80%D0%BE%D0%BA%20-%20%D0%BF%D1%80%D0%B0%D0%B2%D0%B0%20%D0%B8%20%D0%B2%D0%BB%D0%B0%D0%B4%D0%B5%D0%BB%D1%8C%D1%86%D1%8B%20%D0%B2%20multi-user.md).

## Чек-лист выхода

- [ ] git-подмодуль подтянут (каталог lib не пуст)
- [ ] npm-зависимости установлены, кэш Deno внешний
- [ ] config.json валиден и защищен правами 600
- [ ] В логе моста есть starting и волна saved без циклических ошибок
- [ ] Двусторонний тест пройден
