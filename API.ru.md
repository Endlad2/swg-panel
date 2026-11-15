<p align="center"><a href="API.md">English</a> · <b>Русский</b> · <a href="README.technical.md">Technical guide (EN)</a> · <a href="README.technical.ru.md">Техническое руководство (RU)</a></p>

---

# Внешний API интеграций и управления

В **Настройки → Интеграции** теперь можно выпустить API-токен одного из двух типов:

- **Только чтение** — существующий мониторинговый `/api/v1/*` и `/metrics`; менять флот не может.
- **Полное управление** — всё выше плюс те же авторизованные операторские `/api/*` эндпоинты, которыми пользуется SPA.
  Создание пользователей и пиров, управление нодами, подписками, интерфейсами, маршрутизацией и настройками проходит
  через настоящую серверную логику панели: те же проверки, блокировки, журнал событий и побочные эффекты reconcile.

Сырой токен `swgp_…` показывается один раз; на диске остаются только SHA-256-хеш и scope. Токены, выпущенные до
появления scope, остаются **только для чтения**. Выключение API сразу делает недействительными все API-токены.

Мониторинговый read-only слой по-прежнему кэширует общий вид флота примерно на 3 секунды и не создаёт дополнительной
нагрузки на ноды. Управляющие вызовы не идут через этот кэш — они исполняют обычный операторский API панели.

## Содержание

- [Аутентификация](#аутентификация)
- [API управления](#api-управления)
- [Мониторинговые эндпоинты](#мониторинговые-эндпоинты)
- [Webhook'и](#webhookи)
- [Примеры](#примеры)

## Аутентификация

Для обоих scope используйте один из заголовков:

| заголовок | значение |
|---|---|
| `Authorization` | `Bearer <token>` |
| `X-API-Key` | `<token>` |

Форма `?token=<token>` разрешена **только на read-only мониторинговом API** для старых инструментов, которые не умеют
ставить заголовки. Для полного управления токен принимается только из заголовка, чтобы секрет уровня администратора не
попадал в URL, access-логи прокси и историю браузера. Сессионная cookie администратора и HTTP Basic для обычного
операторского API продолжают работать как раньше.

`GET /healthz` и `GET /api/v1/health` — неаутентифицируемые пробы живучести и секретов не раскрывают.

## API управления

Токен со scope **Полное управление** может вызывать существующий операторский API под `/api/*`. Формы запросов и
ответов те же, что использует SPA, поэтому нет отдельного CRUD-слоя, который мог бы разойтись с панелью. Read-only
токен на операторском эндпоинте получает `403` с кодом `read_only_token`; неверный/выключенный токен — `401`.

Часто нужные маршруты:

| метод | путь | назначение |
|---|---|---|
| `GET` | `/api/state` | полное состояние панели, которое читает SPA |
| `GET` | `/api/roster` | ростер пользователей, пиров и групп |
| `POST` | `/api/users/create` | создать пользователя (`{"name":"Alice","tag":"","note":""}`) |
| `POST` | `/api/users/update` / `/api/users/delete` | изменить / удалить пользователя |
| `GET` | `/api/next-ip?nodes=<id>&iface=<name>` | получить следующий свободный клиентский адрес |
| `POST` | `/api/peers/create` | создать пир и развернуть его на одной или нескольких целях |
| `POST` | `/api/peers/update`, `/api/peers/add-target`, `/api/peers/remove-target` | менять развёртывание пира |
| `POST` | `/api/peers/unassign`, `/api/peers/rekey`, `/api/peers/delete` | владение и жизненный цикл учётных данных |
| `GET` | `/api/sub/users` | состояние подписок + зашифрованный escrow |
| `POST` | `/api/sub/user/enable`, `/rotate`, `/disable` | жизненный цикл подписки (те же зашифрованные payload, что у SPA) |
| `GET` / `POST` | `/api/nodes`, `/api/nodes/create`, `/api/nodes/update`, … | управление нодами |
| `POST` | `/api/panel/settings` | настройки панели |
| `POST` | `/api/iface/*`, `/api/turn/*`, `/api/routing/*`, `/api/access/*` | те же операционные действия, что доступны в UI |

Секреты подписки по архитектуре проекта генерируются и шифруются в браузере, поэтому панель **не хранит сырой токен
подписки**. `GET /api/sub/users` отдаёт состояние подписок, но не может раскрыть существующий сырой URL. Для enable/rotate
через операторский API нужно передать те же `token_sha`, `token_by_sk` и escrow-данные, которые формирует SPA. Так
сохраняется существующее правило: plaintext-секрет подписки на сервере не лежит.

### Примеры управления

Создать пользователя:

```bash
curl -sS -H 'Authorization: Bearer swgp_…' \
  -H 'Content-Type: application/json' \
  -d '{"name":"Alice","tag":"team-a"}' \
  https://panel.example.com/api/users/create | jq
```

Прочитать авторитетный ростер:

```bash
curl -sS -H 'X-API-Key: swgp_…' https://panel.example.com/api/roster | jq
```

Создать пир после выбора ноды/интерфейса и получения свободного адреса:

```bash
curl -sS -H 'Authorization: Bearer swgp_…' \
  -H 'Content-Type: application/json' \
  -d '{"user_id":"u_…","pubkey":"<client-public-key>","targets":[{"node":"node-id","iface":"awg0","ip":"10.8.0.10"}]}' \
  https://panel.example.com/api/peers/create | jq
```

## Мониторинговые эндпоинты

### `GET /healthz`
Текстовое `ok` — для проб доступности / балансировщиков. Без авторизации.

### `GET /api/v1/health`
Живучесть + грубые счётчики. Без авторизации.
```json
{ "status": "ok", "version": "1.8.4-beta",
  "nodes": { "total": 5, "online": 5 },
  "peers": { "total": 27, "online": 20 } }
```

### `GET /api/v1/servers`
Каждый узел со статусом, счётчиками и пропускной способностью. Mesh-интерфейсы исключены — счётчики отражают клиентов.
```json
{ "servers": [
  { "id": "1b8e0bcb0b4c", "name": "moscow-1",
    "status": "online", "online": true,          // status: online | offline | never_seen
    "kind": "baremetal", "version": "1.8.4-beta",
    "hostname": "moscow-1", "endpoint_host": "203.0.113.11", "routing_mode": "kernel",
    "interfaces": ["awg0"],
    "peers": 10, "peers_online": 8,
    "rx_bytes_per_sec": 1096290, "tx_bytes_per_sec": 658712,
    "last_seen": 1783620087, "last_seen_age_s": 4,
    "cpu_percent": 59.5, "cpu_max_percent": 97.0, "cpu_saturated_cores": 1, "cpu_cores": [97, 22],
    "cpu_iowait_percent": 1.4, "mem_percent": 52.8, "disk_percent": 45.0, "uptime_s": 3543645 } ] }
```

### `GET /api/v1/servers/{id}`
Один сервер (принимает **id или имя** узла). Форма та же, что у элемента `/servers`.

### `GET /api/v1/servers/{id}/peers`
Пиры, наблюдаемые на этом узле, с **таймингом последнего handshake**, обогащённые данными из ростера.
```json
{ "node": "1b8e0bcb0b4c", "name": "moscow-1", "peers": [
  { "peer_id": "p_9d40f5665d", "public_key": "1+5Gf…=", "iface": "awg0",
    "user": "bob", "title": null, "address": "10.8.0.10",
    "online": true, "endpoint": "203.0.113.9:51820",
    "last_handshake": 1783620085, "handshake_age_s": 2,
    "rx_bytes": 1771968226, "tx_bytes": 1757010049,
    "rx_bytes_per_sec": 413322, "tx_bytes_per_sec": 247993 } ] }
```

### `GET /api/v1/peers`
Идентичности пиров по всему ростеру, с присутствием по каждой цели. Пир считается `online`, если он онлайн
на **любом** из узлов, куда он развёрнут (это совпадает с логикой «самый живой побеждает», которую панель применяет к многоузловым пирам).
```json
{ "peers": [
  { "peer_id": "p_9d40f5665d", "public_key": "1+5Gf…=", "title": null,
    "user": "bob", "user_id": "u_…", "online": true, "last_handshake": 1783620085,
    "targets": [ { "node": "1b8e0bcb0b4c", "iface": "awg0", "address": "10.8.0.10",
                   "online": true, "node_live": true } ] } ] }
```

### `GET /api/v1/summary`
Итоги по флоту.
```json
{ "version": "1.8.4-beta",
  "nodes": { "total": 5, "online": 5 }, "peers": { "total": 27, "online": 20 },
  "throughput": { "rx_bytes_per_sec": 2082447, "tx_bytes_per_sec": 1249463 },
  "generated_at": 1783620181 }
```

### `GET /metrics`
Текстовая экспозиция Prometheus (`v0.0.4`). **Кардинальность только по узлам** — никогда по пирам, что
взорвало бы число серий на большом флоте (детализация по пирам живёт в JSON API). Серии:

| метрика | тип | метки | значение |
|---|---|---|---|
| `swg_panel_up` | gauge | — | 1, когда панель отвечает |
| `swg_panel_build_info` | gauge | `version` | информация о сборке (значение 1) |
| `swg_nodes_total` / `swg_nodes_online` | gauge | — | счётчики узлов флота |
| `swg_peers_total` / `swg_peers_online` | gauge | — | счётчики пиров флота |
| `swg_fleet_rx_bytes_per_second` / `…_tx_…` | gauge | — | суммарная пропускная способность флота |
| `swg_node_up` | gauge | `node`,`name` | 1 — онлайн / 0 — офлайн |
| `swg_node_peers` / `swg_node_peers_online` | gauge | `node`,`name` | счётчики пиров по узлу |
| `swg_node_rx_bytes_per_second` / `…_tx_…` | gauge | `node`,`name` | пропускная способность узла |
| `swg_node_last_seen_timestamp_seconds` | gauge | `node`,`name` | последняя синхронизация (unix) |
| `swg_node_cpu_percent` / `_memory_percent` / `_disk_percent` | gauge | `node`,`name` | здоровье хоста |
| `swg_node_cpu_max_percent` | gauge | `node`,`name` | самый загруженный логический CPU |
| `swg_node_cpu_saturated_cores` | gauge | `node`,`name` | число логических CPU ≥ 90% |
| `swg_node_cpu_iowait_percent` | gauge | `node`,`name` | время ожидания I/O — это **не** CPU |
| `swg_node_uptime_seconds` | gauge | `node`,`name` | аптайм хоста |

`swg_node_cpu_percent` — это **утилизация CPU, усреднённая по логическим CPU (0–100)**: полностью загруженный
узел с 8 CPU показывает `100`, а не `800`, поэтому значение напрямую сравнимо между узлами разного размера.
`swg_node_cpu_max_percent` и `swg_node_cpu_saturated_cores` ловят то, что среднее прячет: однопоточный
датапат насыщает один CPU, и на 8 CPU это среднее `12,5`. Настраивайте алерт на
`swg_node_cpu_saturated_cores > 0`, а не на среднее.

`swg_node_cpu_iowait_percent` — доля времени в ожидании I/O. Она **поднимает load average, но не является
CPU**: именно алерт на один только load и заставляет занятый диск выглядеть занятым процессором.

> **Изменено в 1.2.12-beta.** Раньше `swg_node_cpu_percent` отдавал *load average на ядро*, который считает и
> задачи в непрерываемом сне — так что дисковый I/O и короткий шторм форков (вход по ssh) оба читались как
> «CPU», а значение могло превышать 100. Пороги алертов, настроенные под старый смысл, стоит пересмотреть.
> Узел, всё ещё работающий на старом `swg-noded`, отдаёт прежнее значение load-на-ядро, пока его не обновят,
> и не публикует ни `swg_node_cpu_saturated_cores`, ни `swg_node_cpu_iowait_percent`.

## Webhook'и

Панель отправляет POST'ом подписанное JSON-тело на каждый настроенный URL, когда срабатывает подписанное
событие. События: `peer.added`, `peer.removed`, `node.online`, `node.offline`.

```json
{ "event": "node.offline", "ts": 1783620181,
  "data": { "id": "1b8e0bcb0b4c", "name": "moscow-1", "last_seen": 1783620090 } }
```

Каждая доставка несёт заголовок подписи, чтобы вы могли проверить, что она пришла от панели:

```http
X-SWG-Signature: sha256=<hex HMAC-SHA256(secret, raw_body)>
```

Секрет генерируется при добавлении webhook'а (показывается один раз). Доставка — best-effort с одной
повторной попыткой; онлайн/офлайн узла выводится из давности синхронизации, поэтому `node.offline`
срабатывает, когда узел пропускает синхронизации дольше окна офлайна (`NODE_OFFLINE`, 30 с). Наблюдатель
онлайна/офлайна **не делает вообще ничего**, если webhook'ов не настроено.

## Примеры

**Живучесть** — без авторизации:

```bash
curl -s https://panel.example.com/api/v1/health
```

**Весь флот** — с токеном:

```bash
curl -s -H 'Authorization: Bearer swgp_…' https://panel.example.com/api/v1/servers | jq
```

**Пиры одного узла**, с таймингом handshake:

```bash
curl -s -H 'Authorization: Bearer swgp_…' https://panel.example.com/api/v1/servers/moscow-1/peers | jq
```

**Prometheus** — конфигурация скрейпа:

```yaml
scrape_configs:
  - job_name: swg-panel
    metrics_path: /metrics
    authorization:
      credentials: swgp_…            # ваш API-токен
    static_configs:
      - targets: ['panel.example.com']
```

Uptime Kuma: добавьте монитор **HTTP(s) - Keyword** на `/api/v1/health`, ключевое слово `"ok"`.

Terraform/Ansible: используйте read-only токен для инвентаря/проверки дрейфа, либо токен «Полное управление», если автоматизация должна создавать и менять объекты через тот же операторский API, что UI панели.
