# 02. Docker и Traefik: публикация сервисов

Предыдущая: [01-подготовка vps и безопасность](01-%D0%BF%D0%BE%D0%B4%D0%B3%D0%BE%D1%82%D0%BE%D0%B2%D0%BA%D0%B0%20vps%20%D0%B8%20%D0%B1%D0%B5%D0%B7%D0%BE%D0%BF%D0%B0%D1%81%D0%BD%D0%BE%D1%81%D1%82%D1%8C.md). Следующая: [03-синхронизация obsidian](03-%D1%81%D0%B8%D0%BD%D1%85%D1%80%D0%BE%D0%BD%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F%20obsidian.md).

## Установка Docker

```bash
curl -fsSL https://get.docker.com | sudo bash
```

Работайте с Docker через sudo. Не добавляйте в группу docker пользователя brain: членство в этой группе равно правам root и сломает изоляцию агентов.

## Стек на сервис

Каждый сервис живет в отдельном каталоге со своим compose.yml:

```
/opt/stacks/traefik/compose.yml
/opt/stacks/service-a/compose.yml
/opt/stacks/service-b/compose.yml
```

Сервисы связывает одна внешняя docker-сеть, через которую Traefik видит контейнеры. Создайте ее один раз:

```bash
sudo docker network create traefik_network
```

Так сервисы обновляются и удаляются независимо, а авария одного compose не задевает соседей.

## Traefik: единая точка входа с автоматическим TLS

Создайте файл /opt/stacks/traefik/compose.yml. Замените you@example.com на свой адрес:

```yaml
services:
  traefik:
    image: traefik:v3
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.mytlschallenge.acme.tlschallenge=true
      - --certificatesresolvers.mytlschallenge.acme.email=you@example.com
      - --certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
    networks:
      - traefik_network

networks:
  traefik_network:
    external: true
```

Имя резолвера (здесь mytlschallenge) одинаково в Traefik и в labels каждого сервиса. Несовпадение имен дает ошибку nonexistent certificate resolver.

## Шаблон публикации сервиса

```yaml
services:
  myservice:
    image: vendor/image:tag
    container_name: myservice
    restart: unless-stopped
    env_file:
      - .env
    networks:
      - traefik_network
    labels:
      - traefik.enable=true
      - traefik.http.routers.myservice.rule=Host(`sub.example.com`)
      - traefik.http.routers.myservice.tls=true
      - traefik.http.routers.myservice.tls.certresolver=mytlschallenge
      - traefik.http.routers.myservice.entrypoints=web,websecure
      - traefik.http.services.myservice.loadbalancer.server.port=8080
      - traefik.docker.network=traefik_network

networks:
  traefik_network:
    external: true
```

Обратные кавычки в Host(`...`) - символ слева от цифры 1 на клавиатуре. Обычные кавычки не подходят.

## Порядок публикации сервиса

Выполняйте шаги по порядку. Пропуск шага возвращает на шаг назад.

1. DNS: создайте A-запись поддомена с адресом сервера. Сделайте это до запуска: Let's Encrypt проверяет домен снаружи, без DNS сертификата не будет.
2. Создайте compose.yml по шаблону и файл .env с секретами (права 600).
3. Проверьте файл до запуска:

```bash
sudo docker compose -f /opt/stacks/myservice/compose.yml config --quiet && echo OK
```

4. Запустите:

```bash
sudo docker compose -f /opt/stacks/myservice/compose.yml up -d
```

5. Посмотрите логи Traefik:

```bash
sudo docker logs traefik --since 2m
```

6. Проверьте снаружи: `curl -I https://sub.example.com`. Признак успеха: код ответа 200 или 3xx без ошибки сертификата.

## Порты контейнеров и файрвол

Docker сам создает правила для портов, которые контейнер публикует через `ports:`. Эти правила срабатывают раньше правил UFW. Поэтому опубликованный порт доступен из интернета, даже если UFW его запрещает. Это описано в документации Docker: https://docs.docker.com/engine/network/packet-filtering-firewalls/ (раздел Docker and ufw).

Правила:

1. Через `ports:` публикуйте только Traefik (80 и 443). Остальные сервисы работают за Traefik без `ports:`.
2. Если порт контейнера нужен только на самом сервере, привяжите его к localhost: `"127.0.0.1:5432:5432"`.
3. Порт, нужный контейнерам из docker-сети (например, прокси), открывается правилом UFW для подсети Docker, а не для всех. Пример и разбор: [урок - сеть между хостом и контейнерами](../lessons/%D1%83%D1%80%D0%BE%D0%BA%20-%20%D1%81%D0%B5%D1%82%D1%8C%20%D0%BC%D0%B5%D0%B6%D0%B4%D1%83%20%D1%85%D0%BE%D1%81%D1%82%D0%BE%D0%BC%20%D0%B8%20%D0%BA%D0%BE%D0%BD%D1%82%D0%B5%D0%B9%D0%BD%D0%B5%D1%80%D0%B0%D0%BC%D0%B8.md).
4. После запуска любого сервиса посмотрите, что слушает сервер:

```bash
sudo ss -tlnp
```

Адреса 0.0.0.0 и [::] допустимы только у портов 22, 80, 443 и у портов, которые вы открыли осознанно.

5. Проверьте порт снаружи с устройства, которое не подключено к вашему серверу и его VPN, например с телефона по мобильной сети. В Windows PowerShell: `Test-NetConnection АДРЕС -Port ПОРТ`. В ответе смотрите поле InterfaceAlias: оно показывает, каким путем реально пошел запрос. Признак успеха: TcpTestSucceeded равно False.

## Ошибки Traefik

| Строка в логе | Причина | Что сделать |
|---|---|---|
| field not found, node: XXX | опечатка в имени label | сверить label по буквам с шаблоном |
| illegal rune literal | в Host() вместо обратных кавычек обычные | заменить на обратные кавычки |
| nonexistent certificate resolver | имя резолвера в labels не совпадает с именем в Traefik | использовать одно имя везде |
| unable to obtain ACME certificate | DNS не указывает на сервер | проверить `nslookup` и подождать распространения записи |
| Gateway Timeout в браузере | контейнер подключен к нескольким сетям, и Traefik обращается не в ту | добавить label traefik.docker.network=имя сети |

## Три правила

1. Перед каждым `up` выполняйте `config --quiet`: опечатка в YAML находится за секунду до запуска.
2. Что реально работает, показывает `docker inspect` (разделы Mounts, Cmd, User), а не compose.yml. Файл - это намерение, inspect - состояние. Расхождение означает, что `up` не выполнен после правки. Разбор: [урок - намерение vs реальность (inspect, не compose)](../lessons/%D1%83%D1%80%D0%BE%D0%BA%20-%20%D0%BD%D0%B0%D0%BC%D0%B5%D1%80%D0%B5%D0%BD%D0%B8%D0%B5%20vs%20%D1%80%D0%B5%D0%B0%D0%BB%D1%8C%D0%BD%D0%BE%D1%81%D1%82%D1%8C%20%28inspect%2C%20%D0%BD%D0%B5%20compose%29.md).
3. Данные сервиса с состоянием хранятся только через volume на хосте. После первого запуска создайте файл через сервис и убедитесь, что он появился в папке на хосте. Без этого теста конфигурация не считается примененной. Разбор: [урок - данные в контейнере без volume](../lessons/%D1%83%D1%80%D0%BE%D0%BA%20-%20%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D0%B5%20%D0%B2%20%D0%BA%D0%BE%D0%BD%D1%82%D0%B5%D0%B9%D0%BD%D0%B5%D1%80%D0%B5%20%D0%B1%D0%B5%D0%B7%20volume.md).

## Чек-лист выхода из этапа

- [ ] Traefik запущен и единственный слушает порты 80 и 443
- [ ] Тестовый сервис (например, whoami) опубликован по HTTPS с действующим сертификатом
- [ ] В логе Traefik нет ошибок
- [ ] `sudo ss -tlnp` не показывает лишних портов на 0.0.0.0
