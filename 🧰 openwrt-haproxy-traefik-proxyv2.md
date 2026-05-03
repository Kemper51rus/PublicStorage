# HAProxy (TCP 80/443) → Traefik с PROXY v2 на OpenWrt 25.12.0-rc3 (apk)

> **Цель:** роутер на OpenWrt принимает входящие **HTTP(80)/HTTPS(443) с WAN**, а дальше **пробрасывает TCP** на `100.100.0.5:80/443` (Traefik), передавая реальный IP клиента через **PROXY protocol v2**.

---

## 📌 Предпосылки и важные замечания

- ⚠️ Открывать **LuCI/SSH в WAN** опасно. В этой инструкции LuCI не открываем в WAN (или сильно ограничиваем).
- ✅ Для работы `send-proxy-v2` **Traefik обязан принимать PROXY protocol** на entrypoints `:80` и `:443`.
- ✅ Порты **80/443 на OpenWrt должны быть свободны** (часто их занимает `uhttpd`).

---

## 🗺️ Содержание

- [1. Установка HAProxy (apk)](#1-установка-haproxy-apk)
- [2. Освобождаем 80/443 от uhttpd, не отключая LuCI](#2-освобождаем-80443-от-uhttpd-не-отключая-luci)
- [3. Конфиг HAProxy (готовый)](#3-конфиг-haproxy-готовый)
- [4. Проверка конфига (до запуска)](#4-проверка-конфига-до-запуска)
- [5. Firewall: принимаем 80/443 с WAN](#5-firewall-принимаем-80443-с-wan)
- [6. Traefik: включаем приём PROXY protocol](#6-traefik-включаем-приём-proxy-protocol)
- [7. Запуск и автозапуск](#7-запуск-и-автозапуск)
- [8. Проверка работоспособности](#8-проверка-работоспособности)
- [9. Диагностика проблем](#9-диагностика-проблем)
- [10. Шпаргалка команд](#10-шпаргалка-команд)

---

## 1) 📦 Установка HAProxy (apk)

OpenWrt 25.12 использует пакетный менеджер **apk**.

```sh
apk update
apk -U add haproxy
```

Проверка:
```sh
haproxy -v
apk info haproxy
```

---

## 2) 🚦 Освобождаем 80/443 от uhttpd, не отключая LuCI

### 2.1 🔍 Кто сейчас слушает 80/443?
```sh
ss -lntp | grep -E ':(80|443)\b'
```

Если видите `uhttpd` — выбирайте один из вариантов ниже.

---

### Вариант A ✅ Рекомендовано: перенести LuCI на другие порты (8080/8443)

**Плюсы:** просто, надёжно, HAProxy спокойно забирает 80/443.  
**Минусы:** LuCI будет на других портах.

#### Через UCI (без ручного редактирования)
```sh
# удалить текущие listen_* (на всякий случай)
uci -q delete uhttpd.main.listen_http
uci -q delete uhttpd.main.listen_https

# назначить новые порты
uci add_list uhttpd.main.listen_http='0.0.0.0:8080'
uci add_list uhttpd.main.listen_https='0.0.0.0:8443'

uci commit uhttpd
/etc/init.d/uhttpd restart
```

Теперь LuCI:
- http://ROUTER_IP:8080
- https://ROUTER_IP:8443

---

### Вариант B 🧠 Оставить LuCI на 80/443 только в LAN, а HAProxy — только на WAN

**Идея:**  
- `uhttpd` слушает **только LAN-IP** роутера (например `192.168.1.1:80/443`)  
- HAProxy слушает **только WAN-IP** роутера (например `X.X.X.X:80/443`)

> Это удобно, если вы хотите и дальше заходить в LuCI как раньше (через `http://192.168.1.1`), но при этом отдать **WAN:80/443** под HAProxy.

#### Шаг 1 — привязать uhttpd к LAN-IP
Замените `192.168.1.1` на реальный IP вашего роутера в LAN (обычно `br-lan`).

```sh
uci -q delete uhttpd.main.listen_http
uci -q delete uhttpd.main.listen_https

uci add_list uhttpd.main.listen_http='192.168.1.1:80'
uci add_list uhttpd.main.listen_https='192.168.1.1:443'

uci commit uhttpd
/etc/init.d/uhttpd restart
```

#### Шаг 2 — привязать HAProxy к WAN-IP (в haproxy.cfg)
Вместо `bind *:80` / `bind *:443` используйте `bind WAN_IP:80` и `bind WAN_IP:443`.

Как узнать WAN IPv4:
```sh
WAN_DEV="$(uci -q get network.wan.device || uci -q get network.wan.ifname)"
ip -4 addr show dev "$WAN_DEV"
```

Затем в `/etc/haproxy.cfg`:
```haproxy
frontend http-in
    bind <WAN_IP>:80
    mode tcp
    default_backend traefik-http

frontend https-in
    bind <WAN_IP>:443
    mode tcp
    default_backend traefik-https
```

> ⚠️ Если WAN-IP динамический (PPPoE и т.п.) — после смены IP надо обновить конфиг, либо использовать вариант A.

---

### Вариант C 🧪 (Продвинуто) Привязать HAProxy к интерфейсу (динамический WAN-IP)

HAProxy умеет ограничивать bind конкретным интерфейсом на Linux (опция `interface`), но это зависит от сборки/прав и может быть капризным на некоторых системах.

Пример:
```haproxy
frontend http-in
    bind :80 interface pppoe-wan
    mode tcp
    default_backend traefik-http
```

Если при старте получите `cannot bind socket [0.0.0.0:80]` — используйте вариант A или B.

---

## 3) ⚙️ Конфиг HAProxy (готовый)

Откройте файл:
```sh
vi /etc/haproxy.cfg
```

Вставьте:

```haproxy
global
    log /dev/log local0
    maxconn 2048
    # daemon  # обычно не нужен под init/procd; для диагностики лучше без него

defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend https-in
    bind *:443
    mode tcp
    default_backend traefik-https

frontend http-in
    bind *:80
    mode tcp
    default_backend traefik-http

backend traefik-http
    mode tcp
    server traefik 100.100.0.5:80 send-proxy-v2

backend traefik-https
    mode tcp
    server traefik 100.100.0.5:443 send-proxy-v2
```

> Если используете **Вариант B** (разделение WAN/LAN), замените `bind *:80` / `bind *:443` на `bind <WAN_IP>:80` / `bind <WAN_IP>:443`.

---

## 4) ✅ Проверка конфига (до запуска)

```sh
haproxy -c -f /etc/haproxy.cfg
```

Если есть ошибка — HAProxy напишет строку/причину и не запустится.

---

## 5) 🧱 Firewall: принимаем 80/443 с WAN

Если HAProxy должен принимать входящие с WAN, нужно разрешить **input** на 80/443.

> ⚠️ Важно: это **вход на сам роутер**, не DNAT. Если OpenWrt стоит за другим роутером — там ещё нужен port-forward.

### 5.1 🔓 Разрешить всем (быстро, но менее безопасно)
```sh
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WAN-HTTP-to-HAProxy'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].dest_port='80'
uci set firewall.@rule[-1].target='ACCEPT'

uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WAN-HTTPS-to-HAProxy'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].dest_port='443'
uci set firewall.@rule[-1].target='ACCEPT'

uci commit firewall
/etc/init.d/firewall restart
```

### 5.2 🔒 Рекомендовано: ограничить по IP (если возможно)
Например, только ваш офисный/домашний внешний IP:

```sh
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WAN-HTTP-MyIP'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].dest_port='80'
uci set firewall.@rule[-1].src_ip='<ВАШ_ВНЕШНИЙ_IP>/32'
uci set firewall.@rule[-1].target='ACCEPT'

uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WAN-HTTPS-MyIP'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].dest_port='443'
uci set firewall.@rule[-1].src_ip='<ВАШ_ВНЕШНИЙ_IP>/32'
uci set firewall.@rule[-1].target='ACCEPT'

uci commit firewall
/etc/init.d/firewall restart
```

---

## 6) 🧩 Traefik: включаем приём PROXY protocol

Так как HAProxy отправляет `send-proxy-v2`, Traefik должен принимать ProxyProtocol на entrypoints.

### Пример static config (YAML)
```yaml
entryPoints:
  web:
    address: ":80"
    proxyProtocol:
      trustedIPs:
        - "IP_OPENWRT/32"
  websecure:
    address: ":443"
    proxyProtocol:
      trustedIPs:
        - "IP_OPENWRT/32"
```

> ✅ **trustedIPs** — безопаснее, чем `insecure: true`.  
> Подставьте IP OpenWrt, с которого Traefik видит входящие соединения (LAN/WG/Tailscale).

---

## 7) ▶️ Запуск и автозапуск

```sh
/etc/init.d/haproxy enable
/etc/init.d/haproxy start
/etc/init.d/haproxy status
```

Проверка, что слушает:
```sh
ss -lntp | grep haproxy
```

---

## 8) 🧪 Проверка работоспособности

### 8.1 Проверить доступность backend (Traefik)
```sh
nc -vz 100.100.0.5 80
nc -vz 100.100.0.5 443
```

### 8.2 Проверить, что 80/443 с WAN открыты
С внешней машины/сервиса:
- `curl -I http://ВАШ_WAN_IP/`
- `openssl s_client -connect ВАШ_WAN_IP:443 -servername your.domain` (если TLS passthrough)

### 8.3 Проверить “реальный IP” на стороне Traefik
Если proxyProtocol включён, Traefik сможет видеть корректный client IP и прокидывать его дальше (в логи/headers), в зависимости от вашей конфигурации логирования/forwarded headers.

---

## 9) 🛠️ Диагностика проблем

### 9.1 HAProxy не стартует: быстрый чек
1) Синтаксис:
```sh
haproxy -c -f /etc/haproxy.cfg
```

2) Порты заняты:
```sh
ss -lntp | grep -E ':(80|443)\b'
```

3) Логи:
```sh
logread -e haproxy
logread -e procd
```

### 9.2 Запуск “в лоб” (foreground) — лучший способ увидеть причину
```sh
/etc/init.d/haproxy stop
haproxy -db -f /etc/haproxy.cfg
```

### 9.3 Частые причины
- ❌ **bind failed** → порты заняты (`uhttpd`, другой сервис) → см. раздел 2.
- ❌ **с WAN не открывается** → firewall rule отсутствует → см. раздел 5.
- ❌ **Traefik ругается на мусор/битые данные** → включили `send-proxy-v2`, а Traefik не принимает proxyProtocol → см. раздел 6.
- ❌ **backend недоступен** → нет маршрута/туннеля до `100.100.0.5` → проверяйте `nc`, `ping`, маршруты.

---

## 10) 🧾 Шпаргалка команд

```sh
# Установка
apk update && apk -U add haproxy

# Кто слушает 80/443
ss -lntp | grep -E ':(80|443)\b'

# Проверка конфига
haproxy -c -f /etc/haproxy.cfg

# Запуск/статус
/etc/init.d/haproxy enable
/etc/init.d/haproxy start
/etc/init.d/haproxy status

# Логи
logread -e haproxy
logread -e procd

# Foreground диагностика
/etc/init.d/haproxy stop
haproxy -db -f /etc/haproxy.cfg
```

---

✅ Готово. Если что-то не стартует — пришлите вывод:

```sh
haproxy -c -f /etc/haproxy.cfg
ss -lntp | grep -E ':(80|443)\b'
logread -e procd | tail -n 80
logread -e haproxy | tail -n 80
```
