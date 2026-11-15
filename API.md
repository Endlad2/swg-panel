<p align="center"><b>English</b> · <a href="API.ru.md">Русский</a> · <a href="README.technical.md">Technical guide (EN)</a> · <a href="README.technical.ru.md">Техническое руководство (RU)</a></p>

---

# External Integration & Management API

swg-panel exposes two API-token scopes from **Settings → Integrations**:

- **Read-only** — the existing `/api/v1/*` monitoring surface and `/metrics`. It cannot mutate the fleet.
- **Full management** — everything above, plus the same authenticated operator `/api/*` endpoints the SPA uses.
  User, peer, node, subscription, routing, interface and settings operations therefore run through the panel's real
  validation, locking, event log and node-reconcile side effects instead of a second implementation.

The raw `swgp_…` token is shown once; only its SHA-256 hash and scope are stored. Tokens minted before scope support
remain **read-only**. Turning the API off rejects all API tokens immediately.

The read-only monitoring view is still cached for ~3 seconds and does not add node polling. Management calls are not
served from that cache: they execute the ordinary operator API exactly as the panel does.

## Contents

- [Authentication](#authentication)
- [Management API](#management-api)
- [Monitoring endpoints](#monitoring-endpoints)
- [Webhooks](#webhooks)
- [Examples](#examples)

## Authentication

Use either header for both scopes:

| header | value |
|---|---|
| `Authorization` | `Bearer <token>` |
| `X-API-Key` | `<token>` |

The query form `?token=<token>` is accepted **only by the read-only monitoring surface** for legacy tools that cannot
set headers. Full-management authentication is header-only so an administrator-equivalent secret does not leak into
URLs, proxy access logs or browser history. An admin session cookie / HTTP Basic login also continues to authenticate
the ordinary operator API.

`GET /healthz` and `GET /api/v1/health` are unauthenticated liveness probes and expose no secrets.

## Management API

A token created with **Full management** scope can call the existing operator API under `/api/*`. Responses and request
bodies are the same ones used by the SPA, so existing automation can use the panel directly without a parallel CRUD
layer. A read-only token presented to an operator endpoint gets `403` with code `read_only_token`; an invalid/disabled
token gets `401`.

Common routes:

| method | path | purpose |
|---|---|---|
| `GET` | `/api/state` | full panel state used by the SPA |
| `GET` | `/api/roster` | users, peers and groups roster |
| `POST` | `/api/users/create` | create a user (`{"name":"Alice","tag":"","note":""}`) |
| `POST` | `/api/users/update` / `/api/users/delete` | update / delete a user |
| `GET` | `/api/next-ip?nodes=<id>&iface=<name>` | ask the panel for the next client address |
| `POST` | `/api/peers/create` | create and deploy a peer to one or more targets |
| `POST` | `/api/peers/update`, `/api/peers/add-target`, `/api/peers/remove-target` | manage peer deployment |
| `POST` | `/api/peers/unassign`, `/api/peers/rekey`, `/api/peers/delete` | ownership / credential lifecycle |
| `GET` | `/api/sub/users` | subscription status + encrypted escrow metadata |
| `POST` | `/api/sub/user/enable`, `/rotate`, `/disable` | subscription lifecycle (same encrypted payloads as the SPA) |
| `GET` / `POST` | `/api/nodes`, `/api/nodes/create`, `/api/nodes/update`, … | node management |
| `POST` | `/api/panel/settings` | panel settings |
| `POST` | `/api/iface/*`, `/api/turn/*`, `/api/routing/*`, `/api/access/*` | the same operational controls exposed by the UI |

Because subscription secrets are intentionally generated/encrypted in the browser, the panel does **not** keep the raw
subscription token. `GET /api/sub/users` can list subscription state, but it cannot reveal an existing raw subscription
URL. Enabling/rotating through the operator API therefore requires the same `token_sha`, `token_by_sk` and escrow
material that the SPA supplies. This preserves the project's existing no-plaintext-subscription-secret design.

### Management examples

Create a user:

```bash
curl -sS -H 'Authorization: Bearer swgp_…' \
  -H 'Content-Type: application/json' \
  -d '{"name":"Alice","tag":"team-a"}' \
  https://panel.example.com/api/users/create | jq
```

Read the authoritative roster:

```bash
curl -sS -H 'X-API-Key: swgp_…' https://panel.example.com/api/roster | jq
```

Create a peer after obtaining a node/interface and a free address:

```bash
curl -sS -H 'Authorization: Bearer swgp_…' \
  -H 'Content-Type: application/json' \
  -d '{"user_id":"u_…","pubkey":"<client-public-key>","targets":[{"node":"node-id","iface":"awg0","ip":"10.8.0.10"}]}' \
  https://panel.example.com/api/peers/create | jq
```

## Monitoring endpoints

### `GET /healthz`
Plain-text `ok` — for uptime probes / load balancers. No auth.

### `GET /api/v1/health`
Liveness + coarse counts. No auth.
```json
{ "status": "ok", "version": "1.8.4-beta",
  "nodes": { "total": 5, "online": 5 },
  "peers": { "total": 27, "online": 20 } }
```

### `GET /api/v1/servers`
Every node with status + counts + throughput. Mesh-link interfaces are excluded — counts are client-facing.
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
One server (accepts the node **id or name**). Same shape as an element of `/servers`.

### `GET /api/v1/servers/{id}/peers`
Peers observed on that node, with **last-handshake timing**, enriched from the roster.
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
Roster-wide peer identities with per-target presence. A peer is `online` if it's online on **any** of its
deployed nodes (matching the panel's most-alive-wins view for multi-node peers).
```json
{ "peers": [
  { "peer_id": "p_9d40f5665d", "public_key": "1+5Gf…=", "title": null,
    "user": "bob", "user_id": "u_…", "online": true, "last_handshake": 1783620085,
    "targets": [ { "node": "1b8e0bcb0b4c", "iface": "awg0", "address": "10.8.0.10",
                   "online": true, "node_live": true } ] } ] }
```

### `GET /api/v1/summary`
Fleet totals.
```json
{ "version": "1.8.4-beta",
  "nodes": { "total": 5, "online": 5 }, "peers": { "total": 27, "online": 20 },
  "throughput": { "rx_bytes_per_sec": 2082447, "tx_bytes_per_sec": 1249463 },
  "generated_at": 1783620181 }
```

### `GET /metrics`
Prometheus text exposition (`v0.0.4`). **Per-node cardinality only** — never per-peer, which would
explode the series count on a large fleet (peer detail lives in the JSON API). Series:

| metric | type | labels | meaning |
|---|---|---|---|
| `swg_panel_up` | gauge | — | 1 whenever the panel answers |
| `swg_panel_build_info` | gauge | `version` | build info (value 1) |
| `swg_nodes_total` / `swg_nodes_online` | gauge | — | fleet node counts |
| `swg_peers_total` / `swg_peers_online` | gauge | — | fleet peer counts |
| `swg_fleet_rx_bytes_per_second` / `…_tx_…` | gauge | — | fleet aggregate throughput |
| `swg_node_up` | gauge | `node`,`name` | 1 online / 0 offline |
| `swg_node_peers` / `swg_node_peers_online` | gauge | `node`,`name` | per-node peer counts |
| `swg_node_rx_bytes_per_second` / `…_tx_…` | gauge | `node`,`name` | per-node throughput |
| `swg_node_last_seen_timestamp_seconds` | gauge | `node`,`name` | last sync (unix) |
| `swg_node_cpu_percent` / `_memory_percent` / `_disk_percent` | gauge | `node`,`name` | host health |
| `swg_node_cpu_max_percent` | gauge | `node`,`name` | busiest single logical CPU |
| `swg_node_cpu_saturated_cores` | gauge | `node`,`name` | count of logical CPUs ≥ 90% |
| `swg_node_cpu_iowait_percent` | gauge | `node`,`name` | time waiting on I/O — **not** CPU |
| `swg_node_uptime_seconds` | gauge | `node`,`name` | host uptime |

`swg_node_cpu_percent` is **CPU utilization, meaned across logical CPUs (0–100)** — a fully busy 8-CPU node
reads `100`, not `800`, so it is directly comparable across differently-sized nodes.
`swg_node_cpu_max_percent` and `swg_node_cpu_saturated_cores` catch what a mean hides: a pinned
single-threaded datapath saturates one CPU, and on 8 CPUs that is a mean of `12.5`. Alert on
`swg_node_cpu_saturated_cores > 0` rather than on the mean.

`swg_node_cpu_iowait_percent` is the share of time blocked on I/O. It **raises the load average but is not
CPU** — alerting on load alone is what makes a busy disk look like a busy processor.

> **Changed in 1.2.12-beta.** `swg_node_cpu_percent` previously reported *load average per core*, which
> counts uninterruptible-sleep tasks — so disk I/O and a brief fork storm (an ssh login) both read as
> "CPU", and the value could exceed 100. Alert thresholds tuned against the old meaning should be
> revisited. A node still running an older `swg-noded` falls back to the old load-per-core value until
> it is updated, and emits neither `swg_node_cpu_saturated_cores` nor `swg_node_cpu_iowait_percent`.

## Webhooks

The panel POSTs a signed JSON body to each configured URL when a subscribed event fires. Events:
`peer.added`, `peer.removed`, `node.online`, `node.offline`.

```json
{ "event": "node.offline", "ts": 1783620181,
  "data": { "id": "1b8e0bcb0b4c", "name": "moscow-1", "last_seen": 1783620090 } }
```

Every delivery carries a signature header so you can verify it came from the panel:

```http
X-SWG-Signature: sha256=<hex HMAC-SHA256(secret, raw_body)>
```

The secret is generated when you add the webhook (shown once). Delivery is best-effort with one retry;
node online/offline is derived from sync staleness, so `node.offline` fires when a node misses syncs past
the offline window (`NODE_OFFLINE`, 30s). The online/offline watcher does **no work at all** when you have
no webhooks configured.

## Examples

**Liveness** — no auth:

```bash
curl -s https://panel.example.com/api/v1/health
```

**The whole fleet** — with a token:

```bash
curl -s -H 'Authorization: Bearer swgp_…' https://panel.example.com/api/v1/servers | jq
```

**One node's peers**, with handshake timing:

```bash
curl -s -H 'Authorization: Bearer swgp_…' https://panel.example.com/api/v1/servers/moscow-1/peers | jq
```

**Prometheus** — scrape config:

```yaml
scrape_configs:
  - job_name: swg-panel
    metrics_path: /metrics
    authorization:
      credentials: swgp_…            # your API token
    static_configs:
      - targets: ['panel.example.com']
```

Uptime Kuma: add an **HTTP(s) - Keyword** monitor on `/api/v1/health`, keyword `"ok"`.

Terraform/Ansible: use a read-only token for inventory/drift checks, or a Full management token when the automation must provision or change the fleet through the same operator API as the panel UI.
