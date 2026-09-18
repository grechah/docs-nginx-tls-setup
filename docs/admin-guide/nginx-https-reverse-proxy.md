---
id: nginx-https-reverse-proxy
title: Настройка HTTPS на Nginx с reverse proxy для backend-приложения
sidebar_label: HTTPS и reverse proxy на Nginx
---

# Настройка HTTPS на Nginx с reverse proxy для backend-приложения

Инструкция описывает, как развернуть Nginx перед backend-приложением, включить TLS с самоподписанным сертификатом и проксировать запросы на порт `8080`. После выполнения всех шагов клиенты обращаются к сервису только по HTTPS, а backend остаётся доступен исключительно на локальном интерфейсе.

:::note
Самоподписанный сертификат подходит для внутренних стендов и тестовых окружений. Для production используйте сертификат от доверенного удостоверяющего центра, например выпущенный через Let's Encrypt: порядок настройки server block при этом не меняется, отличаются только значения `ssl_certificate` и `ssl_certificate_key`.
:::

## Схема работы

```mermaid
flowchart LR
    C["Клиент<br/>браузер / curl"] -->|"HTTPS :443<br/>TLS 1.2 / 1.3"| N["Nginx<br/>reverse proxy"]
    N -->|"HTTP :8080<br/>127.0.0.1"| B["Backend-приложение"]
    B -->|"HTTP-ответ"| N
    N -->|"HTTPS-ответ"| C
```

Nginx терминирует TLS: шифрование заканчивается на нём, дальше трафик идёт по локальной сети незашифрованным. Backend узнаёт исходный протокол и IP-адрес клиента из заголовков `X-Forwarded-Proto` и `X-Forwarded-For`, которые Nginx добавляет к запросу.

## Prerequisites

Перед началом убедитесь, что выполнены следующие условия.

| Условие | Как проверить |
|---|---|
| ОС Ubuntu 22.04 LTS или Debian 12 (для других дистрибутивов отличаются только команды пакетного менеджера и пути) | `cat /etc/os-release` |
| Права `sudo` на сервере | `sudo -v` |
| Backend-приложение запущено и слушает порт `8080` | `ss -tlnp \| grep 8080` |
| Порты 80 и 443 не заняты другими сервисами и открыты в межсетевом экране | `ss -tlnp \| grep -E ':80\|:443'`, `sudo ufw status` |
| Установлен OpenSSL | `openssl version` |
| Для сервера определено доменное имя или зафиксирован IP-адрес | `hostname -f`, `ip -4 addr` |

В примерах используются:

- доменное имя `app.example.internal`;
- backend по адресу `127.0.0.1:8080`;
- каталог для сертификатов `/etc/nginx/ssl`.

Замените их на значения своего стенда.

## Шаг 1. Установить Nginx

1. Обновите список пакетов и установите Nginx:

   ```bash
   sudo apt update
   sudo apt install -y nginx
   ```

2. Включите автозапуск и запустите службу:

   ```bash
   sudo systemctl enable --now nginx
   ```

3. Убедитесь, что служба работает:

   ```bash
   systemctl status nginx
   ```

   В выводе должно быть состояние `active (running)`.

4. Откройте в браузере `http://<адрес-сервера>`. Отображается стартовая страница Nginx.

   [Screenshot: стартовая страница «Welcome to nginx!» в браузере]

## Шаг 2. Сгенерировать самоподписанный сертификат

1. Создайте каталог для ключа и сертификата:

   ```bash
   sudo mkdir -p /etc/nginx/ssl
   ```

2. Сгенерируйте пару «закрытый ключ + сертификат» сроком на 365 дней:

   ```bash
   sudo openssl req -x509 -nodes -days 365 \
     -newkey rsa:2048 \
     -keyout /etc/nginx/ssl/app.example.internal.key \
     -out /etc/nginx/ssl/app.example.internal.crt \
     -subj "/C=RU/ST=Moscow/L=Moscow/O=Example/CN=app.example.internal" \
     -addext "subjectAltName=DNS:app.example.internal,IP:127.0.0.1"
   ```

   Назначение ключей команды:

   - `-x509` — выпустить готовый сертификат, а не запрос на подпись (CSR);
   - `-nodes` — не защищать закрытый ключ паролем, иначе Nginx будет запрашивать его при каждом запуске;
   - `-addext "subjectAltName=..."` — указать имена, для которых действует сертификат. Без SAN современные браузеры отклоняют сертификат, даже если поле `CN` заполнено верно.

3. Ограничьте доступ к закрытому ключу:

   ```bash
   sudo chmod 600 /etc/nginx/ssl/app.example.internal.key
   sudo chown root:root /etc/nginx/ssl/app.example.internal.key
   ```

4. Проверьте содержимое сертификата:

   ```bash
   openssl x509 -in /etc/nginx/ssl/app.example.internal.crt -noout -subject -dates -ext subjectAltName
   ```

   [Screenshot: вывод команды openssl x509 с полями Subject, Not Before, Not After и subjectAltName]

## Шаг 3. Настроить server block с TLS и проксированием

1. Создайте файл конфигурации сайта:

   ```bash
   sudo nano /etc/nginx/sites-available/app.example.internal.conf
   ```

2. Добавьте в файл следующую конфигурацию:

   ```nginx
   # Перенаправление HTTP на HTTPS
   server {
       listen 80;
       server_name app.example.internal;
       return 301 https://$host$request_uri;
   }

   server {
       listen 443 ssl;
       http2 on;
       server_name app.example.internal;

       ssl_certificate     /etc/nginx/ssl/app.example.internal.crt;
       ssl_certificate_key /etc/nginx/ssl/app.example.internal.key;

       ssl_protocols       TLSv1.2 TLSv1.3;
       ssl_ciphers         HIGH:!aNULL:!MD5;
       ssl_session_cache   shared:SSL:10m;
       ssl_session_timeout 1d;

       access_log /var/log/nginx/app.access.log;
       error_log  /var/log/nginx/app.error.log;

       client_max_body_size 20m;

       location / {
           proxy_pass http://127.0.0.1:8080;

           proxy_http_version 1.1;
           proxy_set_header Host              $host;
           proxy_set_header X-Real-IP         $remote_addr;
           proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;

           # Поддержка WebSocket, если backend его использует
           proxy_set_header Upgrade    $http_upgrade;
           proxy_set_header Connection "upgrade";

           proxy_connect_timeout 5s;
           proxy_read_timeout    60s;
       }
   }
   ```

   :::note
   Директива `http2 on;` доступна начиная с Nginx 1.25.1. В более ранних версиях используйте запись `listen 443 ssl http2;`. Версию можно посмотреть командой `nginx -v`.
   :::

3. Включите сайт и отключите конфигурацию по умолчанию, чтобы она не перехватывала запросы:

   ```bash
   sudo ln -s /etc/nginx/sites-available/app.example.internal.conf /etc/nginx/sites-enabled/
   sudo rm -f /etc/nginx/sites-enabled/default
   ```

4. Проверьте синтаксис конфигурации и примените её:

   ```bash
   sudo nginx -t
   sudo systemctl reload nginx
   ```

   Успешная проверка выглядит так: `syntax is ok` и `test is successful`.

   [Screenshot: вывод команды sudo nginx -t с успешной проверкой конфигурации]

### Параметры конфигурации

| Параметр | Значение в примере | Назначение |
|---|---|---|
| `listen 443 ssl` | `443 ssl` | Принимать HTTPS-соединения на стандартном порту TLS. |
| `http2 on` | `on` | Включить HTTP/2: меньше задержек при множестве параллельных запросов. |
| `server_name` | `app.example.internal` | Имя, по которому Nginx выбирает этот server block для запроса. |
| `ssl_certificate` | `/etc/nginx/ssl/app.example.internal.crt` | Путь к сертификату (при использовании CA — к файлу с полной цепочкой). |
| `ssl_certificate_key` | `/etc/nginx/ssl/app.example.internal.key` | Путь к закрытому ключу. Доступ только для `root`. |
| `ssl_protocols` | `TLSv1.2 TLSv1.3` | Разрешённые версии TLS. Устаревшие TLS 1.0 и 1.1 отключены. |
| `ssl_ciphers` | `HIGH:!aNULL:!MD5` | Набор шифров: только стойкие, без анонимных и MD5. |
| `ssl_session_cache` | `shared:SSL:10m` | Общий кеш TLS-сессий на 10 МБ, ускоряет повторные подключения. |
| `proxy_pass` | `http://127.0.0.1:8080` | Адрес backend-приложения, куда передаются запросы. |
| `proxy_http_version` | `1.1` | Версия HTTP в сторону backend: нужна для keep-alive и WebSocket. |
| `proxy_set_header Host` | `$host` | Передать backend исходное имя хоста из запроса клиента. |
| `proxy_set_header X-Real-IP` | `$remote_addr` | Передать IP-адрес клиента, иначе backend видит только адрес Nginx. |
| `proxy_set_header X-Forwarded-For` | `$proxy_add_x_forwarded_for` | Цепочка IP-адресов клиента и промежуточных прокси. |
| `proxy_set_header X-Forwarded-Proto` | `$scheme` | Сообщить backend, что исходный запрос пришёл по HTTPS. |
| `proxy_connect_timeout` | `5s` | Таймаут установления соединения с backend. |
| `proxy_read_timeout` | `60s` | Таймаут ожидания ответа от backend. |
| `client_max_body_size` | `20m` | Максимальный размер тела запроса, например при загрузке файлов. |
