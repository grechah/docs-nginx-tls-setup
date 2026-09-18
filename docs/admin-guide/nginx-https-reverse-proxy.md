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

## Шаг 4. Проверить настройку

1. Убедитесь, что Nginx слушает оба порта:

   ```bash
   sudo ss -tlnp | grep nginx
   ```

   Ожидаемый результат — строки с `:80` и `:443`.

2. Отправьте тестовый запрос. Ключ `-k` отключает проверку сертификата, она ожидаемо не проходит для самоподписанного сертификата:

   ```bash
   curl -vk https://app.example.internal/
   ```

   В выводе должны быть строки об установленном соединении `TLSv1.3` и ответ backend-приложения.

3. Проверьте параметры TLS-соединения:

   ```bash
   openssl s_client -connect app.example.internal:443 -servername app.example.internal < /dev/null
   ```

4. Проверьте перенаправление с HTTP на HTTPS:

   ```bash
   curl -I http://app.example.internal/
   ```

   Ожидаемый ответ — `HTTP/1.1 301 Moved Permanently` с заголовком `Location: https://app.example.internal/`.

5. Убедитесь, что backend получает запросы: в его логах отображаются обращения с заголовком `X-Forwarded-Proto: https`.

   [Screenshot: вывод curl -vk с успешным TLS-рукопожатием и ответом backend]

6. Откройте адрес в браузере. Браузер покажет предупреждение о недоверенном сертификате — это ожидаемое поведение для самоподписанного сертификата.

   [Screenshot: предупреждение браузера о недоверенном сертификате с кнопкой перехода на сайт]

## Troubleshooting

### 502 Bad Gateway

**Симптом.** Nginx отвечает `502 Bad Gateway`, в `/var/log/nginx/app.error.log` есть запись `connect() failed (111: Connection refused) while connecting to upstream`.

**Причина.** Backend не запущен, слушает другой порт или только внешний интерфейс, либо соединение блокирует межсетевой экран или SELinux.

**Решение.**

1. Проверьте, что backend слушает нужный адрес: `ss -tlnp | grep 8080`. Если приложение слушает `0.0.0.0:8080`, а не `127.0.0.1:8080`, адрес в `proxy_pass` остаётся рабочим, но при привязке к другому интерфейсу его нужно скорректировать.
2. Запустите backend, если он остановлен, и повторите запрос `curl http://127.0.0.1:8080/` прямо с сервера.
3. В системах с SELinux (RHEL, CentOS, Rocky Linux) разрешите Nginx сетевые подключения: `sudo setsebool -P httpd_can_network_connect 1`.

### Nginx не запускается: адрес уже используется

**Симптом.** Команда `sudo nginx -t` или перезапуск службы завершается ошибкой `bind() to 0.0.0.0:443 failed (98: Address already in use)`.

**Причина.** Порт 443 занят другим процессом или тот же порт объявлен в двух включённых конфигурациях, например в новом файле и в оставшейся конфигурации `default`.

**Решение.**

1. Найдите процесс, занимающий порт: `sudo ss -tlnp | grep ':443'`.
2. Остановите конфликтующий сервис или измените его порт.
3. Проверьте, что порт не объявлен дважды: `grep -r "listen 443" /etc/nginx/sites-enabled/`. Удалите лишнюю ссылку из `sites-enabled` и выполните `sudo systemctl reload nginx`.

### Ошибка при загрузке сертификата или ключа

**Симптом.** Проверка конфигурации завершается ошибкой `SSL_CTX_use_PrivateKey_file(... ) failed (key values mismatch)` или `cannot load certificate ... No such file or directory`.

**Причина.** Пути к файлам указаны неверно, ключ и сертификат из разных пар, либо у процесса Nginx нет прав на чтение ключа.

**Решение.**

1. Проверьте пути и права: `sudo ls -l /etc/nginx/ssl/`.
2. Сравните отпечатки открытого ключа в сертификате и в закрытом ключе — значения должны совпадать:

   ```bash
   sudo openssl x509 -noout -modulus -in /etc/nginx/ssl/app.example.internal.crt | openssl md5
   sudo openssl rsa  -noout -modulus -in /etc/nginx/ssl/app.example.internal.key | openssl md5
   ```

3. Если значения различаются, перевыпустите пару по инструкции из шага 2 и снова выполните `sudo nginx -t`.

## Дальнейшие шаги

- Замените самоподписанный сертификат на сертификат доверенного удостоверяющего центра перед выводом сервиса в production.
- Настройте мониторинг срока действия сертификата: самоподписанный сертификат из этой инструкции истекает через 365 дней.
- Ограничьте доступ к backend на сетевом уровне, чтобы порт `8080` был доступен только с localhost.
