# VLESS REALITY, XHTTP, WebSocket и Hysteria2 на практике

Заметки команды [Vetryx](https://www.vetryx.xyz/) — мы делаем VPN-клиенты для Windows и Android на ядре [Xray](https://github.com/XTLS/Xray-core). Здесь то, что пригодится всем, кто настраивает клиент сам или пишет свой: какие протоколы для чего, как выглядят ссылки подписки, как устроить маршрутизацию «российские сайты напрямую» и на какие грабли мы наступили.

> **English summary.** Practical notes on VLESS + REALITY, VLESS + XHTTP, VLESS + WebSocket (TLS) and Hysteria2 from a team building Xray-based VPN clients for Windows and Android: share-link formats, client config examples, split routing (Russian sites direct), DNS/IPv6 pitfalls and lessons learned. All addresses and keys in the examples are placeholders.

Все адреса, ключи и пароли в примерах — заглушки (`SERVER`, `UUID`, `PUBLIC_KEY`…). Подставьте свои.

## Содержание

1. [Зачем несколько протоколов](#1-зачем-несколько-протоколов)
2. [Форматы ссылок](#2-форматы-ссылок)
3. [Пример клиентского конфига Xray](#3-пример-клиентского-конфига-xray)
4. [Маршрутизация: российские сайты напрямую](#4-маршрутизация-российские-сайты-напрямую)
5. [DNS и IPv6](#5-dns-и-ipv6)
6. [Грабли, на которые мы наступили](#6-грабли-на-которые-мы-наступили)
7. [Мобильный интернет и «белые списки»](#7-мобильный-интернет-и-белые-списки)
8. [Если не подключается: чек-лист](#8-если-не-подключается-чек-лист)

## 1. Зачем несколько протоколов

WireGuard и OpenVPN технически хороши, но у них узнаваемое начало соединения: системы фильтрации трафика распознают рукопожатие по первым пакетам. Поэтому ставка на один протокол — это ставка на то, что его не начнут распознавать именно в вашей сети.

| Протокол | Как выглядит снаружи | Когда нужен |
|---|---|---|
| VLESS + REALITY | TLS 1.3 к реальному известному сайту | Основной вариант. Посторонний, обратившийся к серверу, видит настоящий сайт |
| VLESS + XHTTP | Обычные HTTP-запросы | Другой набор признаков, если REALITY начали резать |
| VLESS + WebSocket (TLS) | Веб-соединение через HTTPS | Запасной вариант для самых строгих сетей, можно ставить за CDN |
| Hysteria2 | QUIC (UDP) | Скорость на нестабильном мобильном интернете |

Клиент не должен заставлять пользователя разбираться в этом. В нашем клиенте в подписке приходят все серверы, а в режиме «Авто» приложение пингует их и выбирает самый быстрый.

## 2. Форматы ссылок

Ссылки такого вида понимают большинство клиентов (v2rayN, v2rayNG, Hiddify, Incy, Happ, Nekoray и др.). Подписка — это обычно base64 со списком таких ссылок по одной в строке.

**VLESS + REALITY**
```
vless://UUID@SERVER:443?encryption=none&flow=xtls-rprx-vision&security=reality&sni=www.example.com&fp=chrome&pbk=PUBLIC_KEY&sid=SHORT_ID&type=tcp#Germany-REALITY
```
- `sni` — домен настоящего сайта, под который маскируется сервер;
- `pbk` — публичный ключ REALITY, `sid` — short id;
- `fp=chrome` — отпечаток TLS-клиента;
- `flow=xtls-rprx-vision` — только для `type=tcp`.

**VLESS + XHTTP**
```
vless://UUID@SERVER:443?encryption=none&security=reality&sni=www.example.com&fp=chrome&pbk=PUBLIC_KEY&sid=SHORT_ID&type=xhttp&path=%2Fxhttp&mode=auto#Germany-XHTTP
```
XHTTP можно сочетать и с REALITY, и с обычным TLS (`security=tls`).

**VLESS + WebSocket (TLS)**
```
vless://UUID@your.domain:443?encryption=none&security=tls&sni=your.domain&fp=chrome&type=ws&host=your.domain&path=%2Fws#Germany-WS
```

**Hysteria2**
```
hysteria2://PASSWORD@SERVER:443?sni=your.domain&obfs=salamander&obfs-password=OBFS_PASSWORD#Finland-Hysteria2
```

Совет: пишите протокол в названии сервера (`#Germany-REALITY`). Когда один протокол перестаёт работать, пользователь сразу видит, на что переключиться.

## 3. Пример клиентского конфига Xray

Полный пример — в файле [`client-vless-reality.json`](client-vless-reality.json): локальный SOCKS/HTTP-прокси, исходящее подключение VLESS + REALITY и маршрутизация из раздела 4. Для Hysteria2 через официальный клиент — [`hysteria2-client.yaml`](hysteria2-client.yaml).

Исходящее подключение VLESS + REALITY:

```json
{
  "tag": "proxy",
  "protocol": "vless",
  "settings": {
    "vnext": [{
      "address": "SERVER",
      "port": 443,
      "users": [{ "id": "UUID", "encryption": "none", "flow": "xtls-rprx-vision" }]
    }]
  },
  "streamSettings": {
    "network": "tcp",
    "security": "reality",
    "realitySettings": {
      "serverName": "www.example.com",
      "fingerprint": "chrome",
      "publicKey": "PUBLIC_KEY",
      "shortId": "SHORT_ID"
    }
  }
}
```

Для XHTTP меняется только `streamSettings`:

```json
"streamSettings": {
  "network": "xhttp",
  "security": "reality",
  "realitySettings": { "serverName": "www.example.com", "fingerprint": "chrome", "publicKey": "PUBLIC_KEY", "shortId": "SHORT_ID" },
  "xhttpSettings": { "path": "/xhttp", "mode": "auto" }
}
```

## 4. Маршрутизация: российские сайты напрямую

Если весь трафик идёт через зарубежный сервер, первыми в поддержку пишут про банки (капча, блокировка входа), госсервисы и маркетплейсы. Поэтому российские ресурсы лучше пускать напрямую. Порядок правил в нашем клиенте — файл [`routing-ru-direct.json`](routing-ru-direct.json):

1. частные адреса (`geoip:private`) — напрямую;
2. свой список российских доменов (у нас около 1500, приходит с сервера) — напрямую;
3. `geosite:category-ru` и `geoip:ru` — напрямую;
4. всё остальное — через прокси.

```json
"routing": {
  "domainStrategy": "IPIfNonMatch",
  "rules": [
    { "type": "field", "ip": ["geoip:private"], "outboundTag": "direct" },
    { "type": "field", "domain": ["domain:gosuslugi.ru", "domain:example-bank.ru"], "outboundTag": "direct" },
    { "type": "field", "domain": ["geosite:category-ru"], "outboundTag": "direct" },
    { "type": "field", "ip": ["geoip:ru"], "outboundTag": "direct" },
    { "type": "field", "network": "tcp,udp", "outboundTag": "proxy" }
  ]
}
```

Чтобы доменные правила срабатывали и для подключений по IP, включите на входящем подключении `sniffing` с `"destOverride": ["http", "tls", "quic"]` и `"routeOnly": true`.

Побочный эффект, который придётся объяснять пользователям: сайты проверки IP из России (2ip.ru и т. п.) при включённом VPN показывают настоящий адрес. Это не поломка — проверять нужно на зарубежном сайте (ipinfo.io, ifconfig.me). Подробнее — в статье [«Как проверить, что VPN работает»](https://www.vetryx.xyz/help/proverit-vpn/).

## 5. DNS и IPv6

```json
"dns": { "servers": ["1.1.1.1", "1.0.0.1"], "queryStrategy": "UseIPv4" }
```

- **IPv6, которого нет.** Если туннель объявляет IPv6, а у провайдера пользователя IPv6 нет, «прямые» сайты начинают зависать: система пробует IPv6-адреса, которые никуда не ведут. Решение — DNS внутри туннеля отдаёт только IPv4 (`UseIPv4`), а IPv6-трафик всё равно перехватывается, чтобы не было утечек.
- **Курица и яйцо.** DNS туннеля сам идёт через прокси. Если адрес сервера задан доменом и резолвится через туннель, туннель не поднимется. Домены серверов резолвьте мимо туннеля.

## 6. Грабли, на которые мы наступили

1. **Ядро ходит само через себя.** В режиме «весь трафик» (TUN на Windows, VpnService на Android) собственный трафик ядра должен идти мимо туннеля, иначе ядро пытается подключиться к серверу через самого себя и всё зависает. На Android — `VpnService.Builder.addDisallowedApplication(свой пакет)` или `protect()` для сокетов ядра.
2. **«Подключено» — ещё не значит «работает».** Сразу после поднятия туннеля проверяйте, что трафик реально проходит. Если нет — выключайте туннель с понятной ошибкой, а не держите статус «Защищено» без интернета. Периодическая проверка связи плюс автопереключение на другой сервер, если интернет в целом есть, а сервер молчит.
3. **Браузер помнит, что было до VPN.** Самая неожиданная история из поддержки: VPN работал в одном профиле Chrome и не работал в основном — даже в режиме TUN. Chrome держит пул соединений и кэш DNS, открытые до включения VPN. Лечится `chrome://net-internals/#sockets` → **Flush socket pools** и `chrome://net-internals/#dns` → **Clear host cache** или перезапуском браузера.
4. **Системный прокси перехватывают.** В режиме системного прокси на Windows другие программы (другие VPN, антивирусы) могут переписать настройки. Клиенту стоит следить за этим и предупреждать пользователя. Отдельный случай — PPPoE и VPN-подключения Windows: у них свои настройки прокси.
5. **Перебор серверов.** Если пользователь выбрал страну, разумно перебрать несколько серверов этой страны; если конкретный сервер — одна попытка, без самодеятельности.

## 7. Мобильный интернет и «белые списки»

Некоторые мобильные операторы в отдельных регионах переводят интернет в режим «белых списков»: открываются только разрешённые, в основном российские, ресурсы. Обычные зарубежные серверы в таком режиме недоступны в принципе, сколько ни меняй протокол. Для этого случая нужна отдельная группа серверов, а в клиенте — отдельная вкладка, чтобы пользователь не перебирал вслепую страну за страной. Подробнее — в статье [«Белые списки на мобильном интернете»](https://www.vetryx.xyz/help/belye-spiski/).

## 8. Если не подключается: чек-лист

1. Обновите подписку — адреса серверов меняются.
2. Попробуйте сервер с другим протоколом (REALITY ↔ XHTTP ↔ WebSocket ↔ Hysteria2).
3. Проверьте дату и время на устройстве — с неверным временем TLS не поднимется.
4. Отключите другие VPN, расширения-прокси и «веб-защиту» антивируса.
5. Сайты не открываются при подключённом VPN — перезапустите браузер (см. раздел 6, п. 3).
6. На мобильном интернете ничего, кроме российских сайтов, не открывается — это «белые списки» (раздел 7).

Подробный разбор по шагам — [«VPN не подключается: что делать»](https://www.vetryx.xyz/help/vpn-ne-podklyuchaetsya/).

---

**Vetryx** — VPN-клиенты для Windows и Android на Xray: [vetryx.xyz](https://www.vetryx.xyz/) · [Справка](https://www.vetryx.xyz/help/) · [Новости в Telegram](https://t.me/vetryxnews) · [X](https://x.com/VetryxVPN)

Текст — [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.ru): можно копировать и переделывать со ссылкой на источник. Примеры конфигов — свободно, без условий.
