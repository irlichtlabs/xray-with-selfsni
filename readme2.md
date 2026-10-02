# Xray: VLESS + XTLS-Vision + TLS с сайтом-маскировкой

Установочный скрипт для VPS: VPN на ядре Xray, который снаружи выглядит как обычный HTTPS-сайт.

- Клиент с правильным UUID получает VPN.
- Браузер, сканер или DPI, «простукивающий» сервер, получает обычный сайт из `/var/www/html` (по HTTP/1.1 или HTTP/2).
- Сертификат Let's Encrypt выпускается и продлевается автоматически.

---

## Как это устроено

```
                         :443 (TLS, настоящий сертификат)
клиент / браузер ───────► Xray
                           │
                           ├─ VLESS с верным UUID ──► VPN (freedom)
                           │
                           └─ всё остальное (фолбэк)
                                ├─ ALPN http/1.1 ──► nginx 127.0.0.1:8080
                                └─ ALPN h2       ──► nginx 127.0.0.1:8081 (h2c)
                                                       │
                                                       └─► /var/www/html

                         :80
браузер ────────────────► nginx ─► /.well-known/acme-challenge/ (продление сертификата)
                                └► 301 на https
```

TLS на 443 терминирует Xray. nginx наружу слушает только порт 80, а сайт отдаёт на localhost. Реальный IP посетителя передаётся в nginx через PROXY protocol.

---

## Требования

- VPS с Debian 11+ или Ubuntu 20.04+, доступ root.
- Домен, A-запись которого указывает на IP сервера (без проксирования через CDN).
- Свободные порты 80 и 443.

---

## Установка

```bash
export domain=example.com
wget -O xray-install.sh <ссылка на скрипт в вашем репозитории>
bash xray-install.sh
```

В конце скрипт выведет ссылку `vless://...` и QR-код основного пользователя.

После установки положите свой сайт в `/var/www/html` (вместо стандартной заглушки nginx). Сайт должен выглядеть правдоподобно: это и есть маскировка.

> ⚠️ **Не запускайте скрипт повторно на рабочем сервере.** Он сгенерирует новый UUID и перезапишет `config.json`, все добавленные пользователи пропадут. Для изменений на живом сервере см. раздел «Обновление существующего сервера».

---

## Команды управления

| Команда | Что делает |
|---|---|
| `mainuser` | Ссылка и QR-код основного пользователя |
| `newuser` | Создать пользователя (с проверкой конфига перед применением) |
| `rmuser` | Удалить пользователя |
| `sharelink` | Выбрать пользователя из списка и вывести его ссылку |
| `userlist` | Список пользователей |

Служебные:

| Команда | Что делает |
|---|---|
| `xray-mklink <uuid> <имя>` | Сгенерировать ссылку для произвольного UUID |
| `xray-apply <файл>` | Проверить конфиг и только при успехе заменить им текущий и перезапустить Xray |

Подсказка по командам сохраняется в `~/help`.

---

## Файлы

| Путь | Назначение |
|---|---|
| `/usr/local/etc/xray/config.json` | Конфиг Xray |
| `/usr/local/etc/xray/.keys` | UUID основного пользователя и домен (права 600) |
| `/usr/local/etc/xray/xray_cert/` | Сертификат и ключ для Xray |
| `/etc/nginx/sites-available/default` | Конфиг nginx (порт 80 + фолбэки 8080/8081) |
| `/var/www/html` | Сайт-маскировка |
| `~/.acme.sh/` | acme.sh, выпуск и продление сертификата |

---

## Настройки клиента

Ссылка из скрипта уже содержит всё нужное. При ручной настройке:

| Параметр | Значение |
|---|---|
| protocol | `vless` |
| address | ваш домен |
| port | `443` |
| encryption | `none` |
| flow | `xtls-rprx-vision` |
| network | `raw` (`tcp`) |
| security | `tls` |
| serverName (SNI) | ваш домен |
| ALPN | `h2`, `http/1.1` |
| fingerprint | `chrome` |
| mux | выключен (несовместим с Vision) |
| ECH | не использовать |

Подходят клиенты на ядре Xray: v2rayN, v2rayNG, v2rayTun, Streisand, Happ и др.

---

## Изменения относительно оригинального скрипта

### Фолбэки и ALPN

- **HTTP/2 для сайта.** Добавлен второй фолбэк `{ "alpn": "h2", "dest": 8081 }`, nginx слушает `8081` с `http2`. Раньше браузер с HTTP/2 мог получить сломанный сайт, а сервер, не умеющий h2, выделялся среди обычных сайтов.
- **ALPN на сервере** теперь `["h2", "http/1.1"]` (массив), а не строка `"http/1.1"`.
- **ALPN в клиентских ссылках** теперь `h2,http/1.1`. Клиент с `fingerprint: chrome` и одним только `http/1.1` в ALPN не похож на настоящий Chrome: несоответствие видно в TLS-отпечатке.
- **SNI добавлен в ссылки явно** (`sni=домен`). Если когда-нибудь в `address` окажется IP, SNI не пропадёт.
- **Реальные IP посетителей сайта.** Фолбэки отправляются с `xver: 1` (PROXY protocol), nginx принимает их через `proxy_protocol` и `real_ip_header`. В логах nginx больше не `127.0.0.1`.

### Исправленные ошибки

- **Продление сертификата не работало.** В `xray-cert-renew` вызывался `$PWD/.acme.sh` (это каталог, а не программа), а строка с `$domain_ecc` была битой (bash искал переменную `domain_ecc`). Теперь используется встроенный механизм acme.sh: `--install-cert ... --reloadcmd`. acme.sh сам продлевает сертификат по своему cron, копирует его для Xray, выставляет права и перезапускает Xray. Свой cron-скрипт больше не нужен.
- **Продление через HTTP-01.** Порт 80 целиком редиректил на https. Добавлен `location /.well-known/acme-challenge/`, отдающий файлы проверки напрямую.
- **Приватный ключ был доступен всем** (`chmod +r`). Теперь `chmod 600`, владелец `nobody` (пользователь, от которого работает служба Xray).
- **Порядок установки.** Xray ставится до выпуска сертификата, nginx настраивается до запроса к Let's Encrypt. Так `reloadcmd` и HTTP-01 проверка срабатывают с первого раза.

### Чистка и надёжность

- `fingerprint` удалён из серверного `tlsSettings`: это клиентский параметр, на сервере он ничего не делает.
- `network: "tcp"` заменён на актуальное имя `raw`.
- Удалены остатки от Reality: `shortsid` в `.keys`, параметр `spx=/` в ссылках.
- Добавлены `minVersion: "1.2"`, `log.loglevel: warning` и `sniffing` с `routeOnly`.
- Проверка, что переменная `domain` задана, до начала установки.
- `newuser` и `rmuser` применяют изменения через `xray-apply`: новый конфиг сначала проверяется `xray run -test`, и только потом заменяет рабочий. Ошибка больше не роняет VPN у всех.
- Временные файлы создаются через `mktemp`, а не `tmp.json` в текущем каталоге.
- Генерация ссылок вынесена в один скрипт `xray-mklink` вместо трёх копий кода.
- `"level": 0` теперь добавляется и новым пользователям.
- Права `600` на `.keys`.

---

## Обновление существующего сервера

Если сервер установлен оригинальным скриптом, **не запускайте новый скрипт**. Внесите изменения вручную.

**1. Конфиг Xray** (`/usr/local/etc/xray/config.json`). В секции инбаунда заменить фолбэки:

```json
"fallbacks": [
    { "dest": 8080, "xver": 1 },
    { "alpn": "h2", "dest": 8081, "xver": 1 }
]
```

В `tlsSettings` удалить `"fingerprint"` и заменить ALPN:

```json
"alpn": ["h2", "http/1.1"],
```

**2. nginx** (`/etc/nginx/sites-available/default`). Привести к виду:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 127.0.0.1:8080 proxy_protocol;
    listen 127.0.0.1:8081 http2 proxy_protocol;
    server_name example.com;

    set_real_ip_from 127.0.0.1;
    real_ip_header proxy_protocol;

    root /var/www/html;
    index index.html;

    add_header Strict-Transport-Security "max-age=63072000" always;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

> `xver` в Xray и `proxy_protocol` в nginx включаются только вместе. Если включить одно без другого, сайт перестанет открываться.

**3. Продление сертификата.** Удалить старую строку из cron (`crontab -e`, строка с `xray-cert-renew`) и один раз выполнить:

```bash
domain=example.com
CERT_DIR=/usr/local/etc/xray/xray_cert
~/.acme.sh/acme.sh --install-cert -d "$domain" --ecc \
    --fullchain-file "$CERT_DIR/xray.crt" \
    --key-file "$CERT_DIR/xray.key" \
    --reloadcmd "chown nobody:nogroup $CERT_DIR/xray.key && chmod 600 $CERT_DIR/xray.key && systemctl restart xray"
```

**4. Проверить и применить:**

```bash
nginx -t && xray run -test -c /usr/local/etc/xray/config.json \
  && systemctl restart nginx xray
```

**5. Клиенты.** Старые конфиги с ALPN `http/1.1` продолжат работать. Обновите ALPN на `h2, http/1.1` и пропишите `serverName` при удобном случае.

---

## Проверка

```bash
# Сайт по HTTP/2 — ожидается "HTTP/2 200"
curl -sI --http2 https://example.com | head -1

# Сайт по HTTP/1.1 — ожидается "HTTP/1.1 200 OK"
curl -sI --http1.1 https://example.com | head -1

# Редирект с 80 — ожидается 301
curl -sI http://example.com | head -1

# Сертификат и дата окончания
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -dates

# Продление сертификата в cron acme.sh
crontab -l | grep acme.sh

# Статус служб
systemctl status xray nginx --no-pager
```

В логах nginx (`/var/log/nginx/access.log`) должны быть реальные IP посетителей, а не `127.0.0.1`.

---

## Решение проблем

| Симптом | Что проверить |
|---|---|
| Сайт не открывается, VPN работает | Совпадают ли `xver: 1` в Xray и `proxy_protocol` в nginx; `nginx -t` |
| Сайт открывается в curl, но не в браузере | Есть ли фолбэк `alpn: h2` → 8081 и `http2` на этом порту в nginx |
| Xray не стартует после продления | Права на ключ: `ls -l /usr/local/etc/xray/xray_cert/` (владелец `nobody`, 600) |
| Сертификат не продлевается | `~/.acme.sh/acme.sh --cron --force` вручную; доступен ли `http://домен/.well-known/acme-challenge/` |
| `newuser` пишет «Ошибка в новом конфиге» | Рабочий конфиг не тронут; ошибка выведена в консоль |
| Клиент подключается, но трафик не идёт | `flow: xtls-rprx-vision` на клиенте, mux выключен, `encryption: none` |

Логи: `journalctl -u xray -n 50 --no-pager`, `/var/log/nginx/error.log`.

---

## Ограничения схемы

- Домен указывает на IP сервера напрямую. При блокировке по IP пропадут и сайт, и VPN. Резервный вариант — дополнительный инбаунд VLESS + XHTTP на отдельном поддомене за CDN.
- Соединения TCP + Vision к зарубежным хостингам в России могут подвергаться «заморозке» после первых десятков килобайт. Поведение зависит от провайдера и региона.
- SSH на том же IP делает сервер заметнее для сканеров. Ограничьте SSH по IP или пускайте его через туннель.
