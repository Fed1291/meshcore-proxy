**English** · [Français](API.fr.md)

# mcproxy HTTP API v1

The mcproxy data API: JSON over HTTPS plus live events (Server-Sent Events), served by the mcproxy
service itself. Any client can use it: a desktop app, a web page (`fetch` / `EventSource`), Home
Assistant, scripts (`curl`). No client gets anything special: everything a client shows comes
through these endpoints.

**Contents**

1. [Quick start](#quick-start)
2. [Conventions](#conventions)
3. [Endpoint index](#endpoint-index)
4. [Endpoint reference](#endpoint-reference)
5. [Live events](#live-events)
6. [Concepts](#concepts)
7. [Where each setting lives](#where-each-setting-lives)

---

## Quick start

### Base URL

```
https://<proxy>:5080/api/v1
```

`<proxy>` is the address of the machine running mcproxy, for example `192.168.1.50`. The examples
below use that address.

### Certificate

The API uses HTTPS with a self-signed certificate the proxy creates on its first start. Check it
once, then pin it:

1. On the proxy machine, `mcproxy api` prints the certificate's SHA-256 fingerprint.
2. On the client, read the fingerprint the server presents and compare:

   ```bash
   openssl s_client -connect 192.168.1.50:5080 </dev/null 2>/dev/null \
     | openssl x509 -noout -fingerprint -sha256
   ```

   (`openssl` prints it in upper case with colons; `mcproxy api` and `GET /api/v1` give it as
   lowercase hex.)
3. Pin it. With `curl`, either skip the check (`-k`, fine on a trusted LAN) or pin the public key:

   ```bash
   # once: the pin of the server's public key
   PIN=$(openssl s_client -connect 192.168.1.50:5080 </dev/null 2>/dev/null \
     | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der \
     | openssl dgst -sha256 -binary | base64)

   curl -k --pinnedpubkey "sha256//$PIN" https://192.168.1.50:5080/api/v1/status
   ```

Apps should pin on first use, like SSH: show the fingerprint, ask the user once, remember it, and
refuse a different certificate later (saying why).

### Token (optional)

When the proxy has an API token set, every request needs it:

```bash
curl -k -H "Authorization: Bearer <token>" https://192.168.1.50:5080/api/v1/status
```

`EventSource` cannot set headers: pass `?token=<token>` in the URL instead (it may be
percent-encoded). Without a token configured, no header is needed.

### First calls

**What is this server?**

```bash
curl -k https://192.168.1.50:5080/api/v1
```

```json
{
  "api": "mcproxy",
  "version": 1,
  "server": "<mcproxy version>",
  "modules": [],
  "tls_fingerprint": "5e8c1f0a9b7d3e2c4a6f8b0d1e3c5a7f9b2d4e6f8a0c1e3b5d7f9a2c4e6b8d0f",
  "endpoints": ["/api/v1/status", "/api/v1/nodes", "/api/v1/messages", "..."]
}
```

**Is the radio connected?**

```bash
curl -k https://192.168.1.50:5080/api/v1/status
```

```json
{
  "server": { "version": "<mcproxy version>", "uptime_secs": 86400, "modules": [] },
  "radio": {
    "connected": true,
    "connected_secs": 3600,
    "link": "ble",
    "node_name": "Base Station",
    "battery_mv": 4100,
    "path_id_bytes": 1,
    "...": "..."
  },
  "queue": { "waiting": 0, "in_flight": null, "held_acks": 0 },
  "...": "..."
}
```

(The full object is under [`GET /api/v1/status`](#get-apiv1status).)

**The two newest messages**

```bash
curl -k "https://192.168.1.50:5080/api/v1/messages?limit=2"
```

```json
{
  "items": [
    {
      "id": 1234,
      "ts": 1793600400,
      "kind": "channel",
      "channel": "Public",
      "channel_id": "3f9a1c7e5b2d4f60",
      "channel_slot": 0,
      "channel_kind": "public",
      "from": "Hilltop",
      "from_key": null,
      "to": null,
      "to_key": null,
      "with": null,
      "outgoing": false,
      "text": "Good signal up here today",
      "text_state": "shown",
      "hops": 2,
      "snr_db": 6.25,
      "rssi_dbm": -98,
      "path": ["7c", "a1"],
      "path_nodes": [],
      "heard_count": 3,
      "echo_of": null,
      "path_id_bytes": 1,
      "client_id": null,
      "delivery": null,
      "scoped": false,
      "region": null,
      "cli": false
    },
    {
      "id": 1233,
      "ts": 1793600100,
      "kind": "dm",
      "channel": null,
      "channel_id": null,
      "channel_slot": null,
      "channel_kind": null,
      "from": "Base Station",
      "from_key": null,
      "to": "Valley Relay",
      "to_key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
      "with": "7c4d2e8f1a3b",
      "outgoing": true,
      "text": "Are you back online?",
      "text_state": "shown",
      "hops": null,
      "snr_db": null,
      "rssi_dbm": null,
      "path": null,
      "path_nodes": null,
      "heard_count": null,
      "echo_of": null,
      "path_id_bytes": null,
      "client_id": "6f1c2a9e-4b7d-4e2a-9c1f-0a3b5d7e9f11",
      "delivery": {
        "state": "delivered", "route": "direct", "timeout_ms": 4200, "rtt_ms": 2650,
        "at": 1793600103, "repeats": 0, "attempt": 1, "attempts_max": 3
      },
      "scoped": false,
      "region": null,
      "cli": false
    }
  ],
  "next": "1233",
  "newer": null
}
```

(`path_nodes` is shortened here; it holds [resolved hops](#resolved-hops).)

### Sending a message

Make one `id` per message the user sends (a UUID fits) and keep it for every HTTP retry of that
message: the same `id` is never sent twice.

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/messages \
  -H "Content-Type: application/json" \
  -d '{"id": "6f1c2a9e-4b7d-4e2a-9c1f-0a3b5d7e9f12", "channel": "Public", "text": "Hello from the hills"}'
```

```json
{
  "sent": true,
  "message_id": 1235,
  "client_id": "6f1c2a9e-4b7d-4e2a-9c1f-0a3b5d7e9f12"
}
```

A DM uses `"to": "Valley Relay"` (a contact's name or the start of its key) instead of `channel`.
See [`POST /api/v1/messages`](#post-apiv1messages).

### Listening to live events

```bash
curl -k -N "https://192.168.1.50:5080/api/v1/events?types=message,message_status"
```

```
event: message
id: 5012
data: {"id":1236,"ts":1793600500,"kind":"channel","channel":"#hills","from":"Hilltop","text":"Anyone on?", ...}
```

In a browser (the certificate must have been accepted once):

```js
const url = "https://192.168.1.50:5080/api/v1/events?types=message,status&token=" + encodeURIComponent(token);
const events = new EventSource(url);
events.addEventListener("message", (e) => showMessage(JSON.parse(e.data)));
events.addEventListener("status", () => refreshStatus());
events.addEventListener("lagged", () => reloadEverything());
```

See [Live events](#live-events).

---

## Conventions

| Topic | Rule |
|---|---|
| Format | JSON only, UTF-8, `Content-Type: application/json`. Field names are `snake_case`. Every record is an object with named fields (`{"ts": .., "packets": ..}`), never a tuple. |
| Units | In the field name: `battery_mv`, `snr_db`, `rssi_dbm`, `noise_dbm`, `temperature_c`, `distance_km`, `uptime_secs`, `size_bytes`, `age_ms`. Values are raw numbers; clients format them. |
| Time | Integer unix seconds (UTC) in fields named `ts` or `*_at`. Durations are `*_secs` (or `*_ms`). |
| Missing values | `null`, never `0` or `""`. A field is present even when `null` (exceptions are noted per endpoint). Nullable integers are JSON numbers or `null`, never strings. |
| Node keys | A node is its public key, 64 lowercase hex characters (`key`). A `{key}` in a path accepts the full key or a unique prefix of at least 2 hex characters. Not hex or too short: `400 bad_request`; no match: `404 not_found`; several matches: `409 ambiguous`. Nodes heard and archived contacts are searched. |
| Node kind | `"chat"`, `"repeater"`, `"room"`, `"sensor"` or `"unknown"` (`"self"` for our own companion in mesh views). |
| Path ids | Lowercase hex strings, one per hop (`["7c", "a1"]`); 1 to 3 bytes each. See [Path id size](#path-id-size). |
| Time range | History endpoints take `?since=<unix>&until=<unix>`, or `?hours=<n>` (from now back; `hours=0` = everything; more than 878400, 100 years: 400, plain text). `since` wins over `hours`. The default `hours` is given per endpoint (usually 24). |
| Paging | Lists answer `{"items": [...], "next": <cursor or null>}`. Pass `?cursor=<next>` for the next page and `?limit=<n>` (default 500, at most 5000, unless the endpoint says otherwise). **Cursors are strings** (`"500"`, `"1233"`); pass them back unchanged. `next: null` = last page. A cursor is a position: a list that changes between two pages (a node heard again moves to the top of `/nodes`) can repeat or skip an item at the seam, so dedupe by key or id. |
| Errors | The HTTP status says what kind; the body says why: `{"error": {"code": "not_found", "message": "no node matches 'a1b2'"}}`. `code` is a stable machine word, `message` is for people. See the table below. |
| Stability | Fields may be **added** within v1. A field is never renamed, removed or given another meaning in v1 (that would be `/api/v2`). Clients ignore fields they do not know, and treat an unknown enum value (a new `link`, `kind`, `type`) as unknown, not as an error. |

**Error statuses**

| Status | Typical `code` | Meaning |
|---|---|---|
| 400 | `bad_request` (or a specific one) | A parameter or body field is missing or wrong. Nothing was done. |
| 401 | `unauthorized` | Token missing or wrong. |
| 403 | `forbidden` | The client's address is not in the proxy's allow list. (Some endpoints also use 403 for `refused` / `not_admin`.) |
| 404 | `not_found` (or a specific one) | No such node, message, channel, or endpoint. |
| 409 | `ambiguous`, `busy`, ... | A key prefix matches several nodes, or the request conflicts with the current state. |
| 500 | `internal` | Unexpected server error. |
| 503 | `unavailable`, `radio_down`, `no_store`, `no_data`, ... | The companion is not connected or did not answer, the message store is off, the database cannot be opened, or the proxy is stopping. |
| 504 | `no_answer`, `unknown` | No answer in time from the mesh or the companion. |

Every endpoint can also answer 401, 403 and 500; endpoints that read the database answer
`503 no_data` when it cannot be opened. These are not repeated in each Errors table, nor are the
`{key}` errors above.

### Transport and access

- **Where:** served by the mcproxy service itself (config section `[api]`: `enabled = true`,
  `listen = "0.0.0.0:5080"`). Each request reads the database through its own read-only connection,
  so API queries never slow down the radio link.
- **HTTPS** by default (`[api] tls = true`): requests and answers are encrypted. On first start the
  proxy creates a self-signed certificate (`api-cert.pem` and `api-key.pem` in `~/.config/mcproxy` of
  the service user, next to the message key, readable by that user only, kept across upgrades). It
  never expires. `mcproxy api` prints its SHA-256 fingerprint and `GET /api/v1` reports it
  (`tls_fingerprint`, `null` without TLS). A certificate of your own:
  `[api] cert = "/path/fullchain.pem"`, `key = "/path/key.pem"`. `tls = false` = plain HTTP (for
  example behind your own reverse proxy). Browsers accept the self-signed certificate once by hand.
- The MeshCore app port (5000) is a different service: plain TCP, the MeshCore companion protocol.
- **Access:** the `[server] allow` address list applies (empty = every address that can reach it).
  Optional `[api] token` (default: none): when set, every request needs
  `Authorization: Bearer <token>` or `?token=`. With neither, the API is open to the network (the
  proxy logs a warning at start).
- **Browsers:** CORS is open (`Access-Control-Allow-Origin: *`). A preflight (`OPTIONS`) allows the
  methods `GET, POST, PUT, PATCH, DELETE, OPTIONS` and the headers `Authorization, Content-Type,
  Last-Event-ID`, cached for 86400 s. With a token set, the token is the protection.
- **Every answer** carries `X-Content-Type-Options: nosniff` and `Cache-Control: no-store`.
- **Privacy:** messages come with their text (ours and others', every channel and DM) for as long
  as the store keeps it. Channel keys and saved login passwords are never served.
- **Slow requests** are logged by the proxy: an answer that took 3 s or more, and a client that
  left before its answer. The query string (which may hold the token) is not logged.
- **Unknown paths** answer `404 not_found` ("no such endpoint").
- **Bodies** must be valid JSON sent with `Content-Type: application/json`; otherwise the request is
  rejected before it reaches the endpoint (400 or 415, with a plain-text body instead of the JSON
  error object). The same holds for a path parameter of the wrong type (a non-numeric message
  `{id}`).

---

## Endpoint index

| Method | Path | What it does |
|---|---|---|
| **Discovery and status** | | |
| GET | [`/api/v1`](#get-apiv1) | Server name, API version, certificate fingerprint |
| GET | [`/api/v1/status`](#get-apiv1status) | Live state: radio, store, contacts, queue, apps |
| **Messages** | | |
| GET | [`/api/v1/messages`](#get-apiv1messages) | Messages received and sent, newest first |
| POST | [`/api/v1/messages`](#post-apiv1messages) | Send a message (or resend one) |
| DELETE | [`/api/v1/messages`](#delete-apiv1messages) | Delete a DM conversation from the store |
| DELETE | [`/api/v1/messages/{id}`](#delete-apiv1messagesid) | Delete one stored message |
| GET | [`/api/v1/messages/{id}/heard`](#get-apiv1messagesidheard) | Every copy of a message's packet we heard |
| **Channels** | | |
| GET | [`/api/v1/channels`](#get-apiv1channels) | Channels known, with their traffic |
| POST | [`/api/v1/channels`](#post-apiv1channels) | Add a channel on the companion |
| PATCH | [`/api/v1/channels/{id}`](#patch-apiv1channelsid) | Set a channel's own region |
| DELETE | [`/api/v1/channels/{id}`](#delete-apiv1channelsid) | Remove a channel from the companion |
| **Nodes** | | |
| GET | [`/api/v1/nodes`](#get-apiv1nodes) | Nodes heard and contacts |
| GET | [`/api/v1/nodes/{key}`](#get-apiv1nodeskey) | One node |
| PATCH | [`/api/v1/nodes/{key}`](#patch-apiv1nodeskey) | Favourite, tracking, "mine", DM path |
| GET | [`/api/v1/nodes/{key}/adverts`](#get-apiv1nodeskeyadverts) | Every advert heard from a node |
| GET | [`/api/v1/nodes/{key}/heard`](#get-apiv1nodeskeyheard) | How we hear a node over time |
| GET | [`/api/v1/nodes/{key}/status`](#get-apiv1nodeskeystatus) | Status answers of a repeater / room |
| GET | [`/api/v1/nodes/{key}/telemetry`](#get-apiv1nodeskeytelemetry) | Telemetry values of a node |
| **Live events** | | |
| GET | [`/api/v1/events`](#get-apiv1events) | Server-Sent Events stream |
| **Mesh and statistics** | | |
| GET | [`/api/v1/mesh/summary`](#get-apiv1meshsummary) | Packet totals for charts |
| GET | [`/api/v1/mesh/packets`](#get-apiv1meshpackets) | Every packet heard, decoded |
| GET | [`/api/v1/mesh/graph`](#get-apiv1meshgraph) | The mesh as a graph of nodes and links |
| GET | [`/api/v1/mesh/delivery`](#get-apiv1meshdelivery) | How our DMs fared, per contact and route |
| GET | [`/api/v1/mesh/hours`](#get-apiv1meshhours) | Traffic by hour of the day |
| GET | [`/api/v1/mesh/replay`](#get-apiv1meshreplay) | Packets of a window with their way through the mesh |
| GET | [`/api/v1/mesh/adverts`](#get-apiv1meshadverts) | Raw advert times of every node |
| GET | [`/api/v1/messages/stats`](#get-apiv1messagesstats) | Message counts per hour, day, channel, sender |
| GET | [`/api/v1/messages/senders`](#get-apiv1messagessenders) | Everyone who wrote |
| **Node actions** | | |
| POST | [`/api/v1/nodes/{key}/login`](#post-apiv1nodeskeylogin) | Log in to a repeater / room / sensor |
| DELETE | [`/api/v1/nodes/{key}/login`](#delete-apiv1nodeskeylogin) | Forget a saved password |
| POST | [`/api/v1/nodes/{key}/cli`](#post-apiv1nodeskeycli) | Run a CLI command on a node |
| GET | [`/api/v1/nodes/{key}/cli`](#get-apiv1nodeskeycli) | Commands sent to a node and their replies |
| GET | [`/api/v1/cli/commands`](#get-apiv1clicommands) | The CLI commands that exist |
| GET | [`/api/v1/nodes/{key}/neighbours`](#get-apiv1nodeskeyneighbours) | A repeater's neighbour table, as last asked |
| POST | [`/api/v1/nodes/{key}/neighbours`](#post-apiv1nodeskeyneighbours) | Ask for the neighbour table now |
| GET | [`/api/v1/nodes/{key}/owner`](#get-apiv1nodeskeyowner) | Firmware, name and owner text, as last asked |
| POST | [`/api/v1/nodes/{key}/owner`](#post-apiv1nodeskeyowner) | Ask for them now |
| POST | [`/api/v1/nodes/{key}/status`](#post-apiv1nodeskeystatus) | Ask a node for its status now |
| POST | [`/api/v1/nodes/{key}/telemetry`](#post-apiv1nodeskeytelemetry) | Ask a node for its telemetry now |
| POST | [`/api/v1/trace`](#post-apiv1trace) | Trace a path, SNR per hop |
| POST | [`/api/v1/nodes/{key}/restore`](#post-apiv1nodeskeyrestore) | Put an archived contact back on the companion |
| GET | [`/api/v1/nodes/{key}/paths`](#get-apiv1nodeskeypaths) | How a node's paths changed |
| **Regions** | | |
| GET | [`/api/v1/regions`](#get-apiv1regions) | Region names known |
| PUT | [`/api/v1/regions`](#put-apiv1regions) | Replace the names added by hand |
| POST | [`/api/v1/regions/fetch`](#post-apiv1regionsfetch) | Ask nearby repeaters for their regions |
| **Settings** | | |
| GET | [`/api/v1/settings`](#get-apiv1settings) | Proxy settings a client may change |
| PATCH | [`/api/v1/settings`](#patch-apiv1settings) | Change them |
| **Logs** | | |
| GET | [`/api/v1/logs`](#get-apiv1logs) | The service log |
| **Companion** | | |
| PATCH | [`/api/v1/companion/settings`](#patch-apiv1companionsettings) | Auto-add settings and path id size |
| POST | [`/api/v1/companion/advert`](#post-apiv1companionadvert) | Send our advert now |
| PATCH | [`/api/v1/companion/region`](#patch-apiv1companionregion) | The companion's default region |
| GET | [`/api/v1/companion/history`](#get-apiv1companionhistory) | Our companion over time |

---

## Endpoint reference

## Discovery and status

### GET /api/v1

Names the server and lists what it offers; needs no database and no radio.

```bash
curl -k https://192.168.1.50:5080/api/v1
```

```json
{
  "api": "mcproxy",
  "version": 1,
  "server": "<mcproxy version>",
  "modules": [],
  "tls_fingerprint": "5e8c1f0a9b7d3e2c4a6f8b0d1e3c5a7f9b2d4e6f8a0c1e3b5d7f9a2c4e6b8d0f",
  "endpoints": [
    "/api/v1/status", "/api/v1/companion/history", "/api/v1/nodes", "/api/v1/nodes/{key}",
    "/api/v1/messages", "/api/v1/channels", "/api/v1/events"
  ]
}
```

| Field | Type | Meaning |
|---|---|---|
| `api` | string | Always `"mcproxy"`. |
| `version` | integer | API version (`1`). |
| `server` | string | The mcproxy version. |
| `modules` | array of strings | Optional modules built into this program, `[]` when none. A client shows a module's parts only when it is listed. |
| `tls_fingerprint` | string or null | SHA-256 of the certificate (DER), lowercase hex; `null` without TLS. |
| `endpoints` | array of strings | Every endpoint path served (path parameters as `{key}` / `{id}`). |

### GET /api/v1/status

The live state of the proxy (what `mcproxy status` shows).

```bash
curl -k https://192.168.1.50:5080/api/v1/status
```

```json
{
  "server": { "version": "<mcproxy version>", "uptime_secs": 86400, "modules": [] },
  "radio": {
    "connected": true,
    "connected_secs": 3600,
    "link": "ble",
    "serial_port": "",
    "host": "",
    "address": "AA:BB:CC:DD:EE:FF",
    "ble_name": "Base Station (AA:BB:CC:DD:EE:FF)",
    "node_name": "Base Station",
    "node_key": "e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4",
    "reconnects": 2,
    "last_down_reason": null,
    "battery_mv": 4100,
    "battery_at": 1793600000,
    "temperature_c": 6.5,
    "temperature_at": 1793600000,
    "frames_from_radio": 1234,
    "frames_to_radio": 567,
    "path_id_bytes": 1,
    "auto_add": {
      "mode": "selected", "chat": true, "repeater": true, "room": false, "sensor": false,
      "overwrite_oldest": true, "max_hops": 0
    }
  },
  "store": {
    "messages": 10, "metadata_rows": 900, "devices": 3, "size_bytes": 1048576,
    "packets": 50000, "node_rows": 3000,
    "contacts_archived": 420, "contacts_archived_removed": 70,
    "retention": {
      "text_days": 365, "metadata_days": 0, "packets_days": 180, "nodes_days": 365, "contacts_days": 0
    }
  },
  "contacts": { "cached": 347, "fresh": true, "max": 350, "prune_days": 0 },
  "queue": {
    "waiting": 0,
    "in_flight": { "cmd": "GET_BATT_AND_STORAGE", "owner": "#1", "age_ms": 120 },
    "held_acks": 0
  },
  "polite": { "enabled": true, "burst": 4, "gap_secs": 3, "waiting": 0, "acks_open": 0, "acks_open_max": 6 },
  "clients": [
    { "id": 3, "address": "192.168.1.20", "app": "MeshCore", "connected_secs": 120,
      "frames_in": 10, "frames_out": 20, "pending": 0 }
  ],
  "tracked": [
    { "key": "a1b2c3d4e5f6", "name": "Hilltop", "result": "ok", "at": 1793600000 }
  ]
}
```

**`server`**

| Field | Type | Meaning |
|---|---|---|
| `version` | string | mcproxy version. |
| `uptime_secs` | integer | Since the service started. |
| `modules` | array of strings | Optional modules built into this program, `[]` when none. |

**`radio`** (the companion)

| Field | Type | Meaning |
|---|---|---|
| `connected` | bool | The companion is connected now. |
| `connected_secs` | integer | Since the link last went up (or down). |
| `link` | string | `"ble"` (Bluetooth), `"usb"` (cable) or `"tcp"` (a companion with Wi-Fi or Ethernet). Treat other values as unknown. |
| `serial_port` | string | The configured USB port; `""` = found automatically, or another link. |
| `host` | string | The configured network address of the companion, `"host"` or `"host:port"` (meaningful with `tcp`). |
| `address` | string | The configured Bluetooth address (meaningful with `ble`). |
| `ble_name` | string | The companion's display name for any link, e.g. `"Base Station (AA:BB:CC:DD:EE:FF)"`, `"Base Station (USB ttyACM0)"`, `"Base Station (TCP 192.168.1.50:5000)"` (the field keeps its old name). |
| `node_name` | string | The companion's node name. |
| `node_key` | string or null | Its public key. |
| `reconnects` | integer | Link reconnects since start. |
| `last_down_reason` | string or null | Why the link last went down. |
| `battery_mv`, `battery_at` | integer or null | Last battery reading and when. |
| `temperature_c`, `temperature_at` | number / integer or null | Last temperature from the companion's history and when. |
| `frames_from_radio`, `frames_to_radio` | integer | Frame counters. |
| `path_id_bytes` | integer or null | Path id size our companion sends with (its path hash mode, 1-3); `null` = firmware before v10 or not read yet. See [Path id size](#path-id-size). |
| `auto_add` | object or null | Which contacts the companion adds by itself; `null` = not read yet. See [Auto-add settings](#auto-add-settings). |

**`store`** (`null` when the message store is off)

| Field | Type | Meaning |
|---|---|---|
| `messages` | integer | Stored message texts. |
| `metadata_rows` | integer | Stored message rows (metadata). |
| `devices` | integer | Devices (apps by address) known to the store. |
| `size_bytes` | integer | Database size. |
| `packets`, `node_rows` | integer | Packets and node history rows stored. |
| `contacts_archived` | integer | Contacts in the [contact archive](#contact-archive-and-other-companions). |
| `contacts_archived_removed` | integer | Of those, how many left the companion's list. |
| `retention.text_days`, `metadata_days`, `packets_days`, `nodes_days`, `contacts_days` | integer | Retention per kind of data, in days (0 = forever). |

The store counts are cached for up to 15 s.

**`contacts`**

| Field | Type | Meaning |
|---|---|---|
| `cached` | integer | Contacts in the proxy's cache of the companion's list. |
| `fresh` | bool | The cache is checked and current (`false` = being refreshed from the radio). |
| `max` | integer or null | How many contacts the companion can hold. |
| `prune_days` | integer | Automatic removal of contacts not heard for this many days (0 = off). |

**`queue`**

| Field | Type | Meaning |
|---|---|---|
| `waiting` | integer | Commands waiting for the companion. |
| `in_flight` | object or null | The command the companion is working on: `cmd` = its name (as `mcproxy watch` shows it), `owner` = who sent it, `age_ms` = since it was sent. `null` = idle. |
| `held_acks` | integer | ACKs kept for apps on port 5000 that are not connected (see [At-least-once delivery](#at-least-once-delivery-and-held-acks)). |

`owner` values (stable): `"#<id>"` = an app (the `id` in `clients`), `"proxy"` = the proxy itself
(health check, history, message reading), `"api"` = this API (`POST /api/v1/messages`, resends,
favourites).

**`polite`** (see [Polite sending](#polite-sending))

| Field | Type | Meaning |
|---|---|---|
| `enabled` | bool | Polite sending is on. |
| `burst` | integer | On-air commands sent at once before pacing starts. |
| `gap_secs` | integer | Gap between paced on-air commands. |
| `waiting` | integer | On-air commands waiting for their turn now. |
| `acks_open`, `acks_open_max` | integer | DMs waiting for their ACK now, and the most at once. |

**`clients`** (apps connected on the MeshCore port)

| Field | Type | Meaning |
|---|---|---|
| `id` | integer | Connection id (as in `in_flight.owner`). |
| `address` | string | The app's IP address. |
| `app` | string or null | The app's name, once it said it. |
| `connected_secs` | integer | Since it connected. |
| `frames_in`, `frames_out` | integer | Frames from / to it. |
| `pending` | integer or null | Messages waiting for that device in the store. |

**`tracked`** (nodes asked for status and telemetry regularly)

| Field | Type | Meaning |
|---|---|---|
| `key` | string | The tracked entry as saved (a key prefix). |
| `name` | string | Its name (the entry itself when unknown). |
| `result` | string | Outcome of the last round, readable text (`"waiting for the first round"` before the first). |
| `at` | integer or null | When that result came; `null` before the first round. |

| Status | Code | When |
|---|---|---|
| 503 | `unavailable` | The proxy is stopping. |

## Messages

### GET /api/v1/messages

Messages received and sent, **newest first**, paged by message id.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Time range (see [Conventions](#conventions)). |
| `kind` | string | all | `dm` or `channel`. |
| `channel` | string | all | A channel id; else a name (every channel of that name); else a slot number (the channel in it now). See [Channel ids and slots](#channel-ids-and-slots). |
| `with` | hex string | none | One DM conversation, **both ways**: a contact's key or its start (at least 4 hex). Filtered before paging. |
| `from` | string | none | Received from this sender only: a name (any case) or a key prefix. |
| `limit` | integer | 200 | 1 to 2000. |
| `cursor` | string | none | Continue below this message id (from `next`). |
| `after` | integer | none | Serve the page **right above** this message id instead (the `limit` messages just newer than it), still newest first. |

`channel` and `with` are applied in the query itself, so a quiet channel's pages are as fast as a
busy one's. `with` combines with `cursor` and `after`.

**Scrolling both ways.** Without `after`: `next` = continue below (pass it as `cursor`), `newer` is
`null`. With `after`: `newer` = continue above (pass it as `after` again; `null` when the page
reaches the newest), `next` is `null`. A client keeps a window of a few pages: `cursor=<oldest
shown>` loads older, `after=<newest shown>` loads newer, the far end is dropped. New messages arrive
as `message` events: a window at the newest adds them, a window scrolled back ignores them and reads
them with `after` when it comes down.

```bash
curl -k "https://192.168.1.50:5080/api/v1/messages?with=7c4d&limit=50"
```

```json
{
  "items": [
    {
      "id": 1240,
      "ts": 1793600900,
      "kind": "dm",
      "channel": null,
      "channel_id": null,
      "channel_slot": null,
      "channel_kind": null,
      "from": "Valley Relay",
      "from_key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
      "to": null,
      "to_key": null,
      "with": "7c4d2e8f1a3b",
      "outgoing": false,
      "text": "Back online, battery replaced",
      "text_state": "shown",
      "hops": 1,
      "snr_db": 7.5,
      "rssi_dbm": -91,
      "path": ["a1"],
      "path_nodes": [
        {
          "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
          "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
          "name": "Hilltop", "kind": "repeater", "lat": 12.3, "lon": 45.6,
          "confidence": 0.9, "choices": 1
        }
      ],
      "heard_count": 1,
      "echo_of": null,
      "path_id_bytes": 1,
      "client_id": null,
      "delivery": null,
      "scoped": false,
      "region": null,
      "cli": false
    }
  ],
  "next": null,
  "newer": null
}
```

| Field | Type | Meaning |
|---|---|---|
| `id` | integer | Stored message id. |
| `ts` | integer | When the proxy received it (ours: when it was stored). |
| `kind` | string | `"dm"` or `"channel"`. |
| `channel` | string or null | The channel's name (from its id, whatever slot it is in now). `null` when `channel_id` is, or the name is not known. |
| `channel_id` | string or null | 16 hex. Group a conversation by this. `null` on a DM, and on a channel message whose channel cannot be told (see [Channel ids and slots](#channel-ids-and-slots)). |
| `channel_slot` | integer or null | The slot its channel has on the companion now; `null` = not on it, or `channel_id` is `null`. |
| `channel_kind` | string or null | `"public"`, `"hashtag"` (`#name`) or `"private"` (also when the name is unknown); `null` when `channel_id` is. |
| `from` | string | DM: the contact's name (or key prefix). Channel: the name the sender put in front of the text. Ours: our node name. |
| `from_key` | string or null | The full key (64 hex): of a DM's sender when it is a known contact, of a channel sender whose name matches exactly one node. `null` when not known, and on ours. |
| `to` | string or null | DMs we sent: the contact it went to (its name, or the start of its key when not known). |
| `to_key` | string or null | DMs we sent: that contact's full key (64 hex), `null` when not known. Missing on proxies before 0.2.1. |
| `with` | string or null | DMs, both ways: the first 6 bytes (12 hex) of the other side's key, always set, also for someone who is not a contact. **Group DM conversations by this, never by a name** (two contacts can have the same one); it can be given as `?with=`. `null` on channel messages. Missing on proxies before 0.2.1. |
| `outgoing` | bool | Sent by us (from any app or `POST /api/v1/messages`). |
| `text` | string or null | Always set (ours too), `null` once the text has expired or cannot be read. |
| `text_state` | string | `"shown"`, `"expired"` (deleted by retention; the metadata stays), `"data"` (a channel datagram: binary data an app sent on the channel, not text; see `data`) or `"unreadable"` (stored, but the encryption key is not available). Missing on older proxies. |
| `data` | object or null | A channel datagram (`text_state` `"data"`): `{"type": 65535, "hex": "0a1b2c"}`. `type` = the data type the sending app gave (0-65535; MeshCore reserves 0, 65535 is for experiments); `hex` = its bytes. `null` for every text message. |
| `hops` | integer or null | Hop count of the packet. `null` on our own messages (show `delivery.route` instead). |
| `snr_db` | number or null | SNR of the received message. |
| `rssi_dbm` | integer or null | RSSI of the first copy of its packet heard (received messages). |
| `path` | array or null | **The sender's path**: ids as in the packet, from the first copy heard. See [Message paths](#message-paths). |
| `path_nodes` | array or null | Each id of `path` [resolved](#resolved-hops), same order, with the sender as context when known. |
| `heard_count` | integer or null | How many times we heard the packet that carried it; for our channel messages, how many repeats of it we heard (an echo included). `null` = the packet was not matched (DMs we sent: always). |
| `echo_of` | integer or null | On a received channel message that is **our own message coming back** through the mesh: the `id` of our message. Hide it or fold it into ours. Exact (packet fingerprint). From 0.2.1 such a copy is not stored any more: only on older rows. |
| `path_id_bytes` | integer or null | The path id size its packet really had, from a heard copy (ours: the size the companion sent it with). `null` until a copy was heard; for ours, show `/status` `radio.path_id_bytes` then. |
| `client_id` | string or null | For messages sent through `POST /api/v1/messages`, the `id` the client gave it. |
| `delivery` | object or null | Messages we sent: see [Delivery states](#delivery-states-and-dm-retries). `null` for received ones and before the companion answered. |
| `scoped` | bool | Received: its packet was sent in a region (flood scope). |
| `region` | string or null | Received: that region's name, when it matched a known one. |
| `cli` | bool | A node CLI command or reply sent by an app through port 5000, not a chat message: keep it out of chats, unread counts and notifications. |

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | `kind` not `dm` / `channel`; `with` not hex or shorter than 4. |
| 404 | `not_found` | `channel` matches no channel. |
| 409 | `ambiguous` | Several conversations' contact keys start with `with`. |

### POST /api/v1/messages

Sends a message through the companion. **No duplicates by design**: see
[Sending without duplicates](#sending-without-duplicates).

**Body: a new message**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `id` | string | yes | The client's id for this message: letters, digits, `- _ . :`, at most 64 characters (a UUID fits). Made once per message the user sends, never per HTTP attempt. |
| `text` | string | yes | Not empty, no NUL character. At most 160 bytes for a DM; on a channel 160 minus the companion's `"<name>: "` prefix. |
| `channel` | string or integer | one of `channel` / `to` | The channel's id (as `GET /api/v1/channels` gives it; preferred: it is the same whatever slot the channel sits in, so an out-of-date list of slots reaches the right channel or gets `not_found`, never another one), its name (any case) or its slot number. |
| `to` | string | one of `channel` / `to` | A contact's exact name (any case) or the start of its public key (hex, at least 4). |
| `route` | string | no | DMs only: `"auto"` (default: the companion's stored path, flood when none), `"flood"` (the stored path is forgotten first), `"path"` (with `path`; stored first). Like the MeshCore app's long press on Send, the companion keeps the change. Resends still flood before the last attempt. |
| `path` | array of hex strings | with `route: "path"` | Repeater ids, first hop first, 1-3 bytes each, all the same size; `[]` = direct. |

**Body: a resend** (see [Resending](#resending))

| Field | Type | Required | Meaning |
|---|---|---|---|
| `id` | string | yes | A **new** client id for this try. |
| `resend_of` | integer | yes | The stored message id to send again. Goes without `text`, `to` and `channel`. |
| `route` | string | no | As above (a channel resend: only `auto`). |

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/messages \
  -H "Content-Type: application/json" \
  -d '{"id": "b7e2c9d4-1f3a-4c5e-8d7b-2a4c6e8f0a13", "to": "Valley Relay", "text": "See you at the ridge", "route": "auto"}'
```

**Sent** (HTTP 202):

```json
{
  "sent": true,
  "message_id": 1241,
  "client_id": "b7e2c9d4-1f3a-4c5e-8d7b-2a4c6e8f0a13"
}
```

**Not sent** (it never went out; safe to send again with the same `id`):

```json
{
  "sent": false,
  "code": "not_sent",
  "reason": "not_contact",
  "error": "\"Valley Relay\" is not a contact of the companion - a DM needs it in the contacts",
  "client_id": "b7e2c9d4-1f3a-4c5e-8d7b-2a4c6e8f0a13"
}
```

**Unknown** (HTTP 504; it was handed to the companion but no answer came, so it may have gone out):

```json
{
  "sent": false,
  "code": "unknown",
  "error": "the companion did not answer",
  "message_id": 1243,
  "client_id": "b7e2c9d4-1f3a-4c5e-8d7b-2a4c6e8f0a13"
}
```

| Field | Type | Meaning |
|---|---|---|
| `sent` | bool | It went out. |
| `message_id` | integer or null | The stored message (`null` when the message store is off). For a resend, the stored id that was resent. Also with `unknown`: the message is stored with its delivery state `"unknown"` and can be sent again with `resend_of` (missing when the proxy itself did not answer in time, and on proxies before 0.2.1). |
| `client_id` | string | The `id` of the request (every answer carries it). |
| `code` | string | `"not_sent"` or `"unknown"`. |
| `reason` | string | With `not_sent`: why (see below). |
| `error` | string | Readable explanation. |

The answer comes once the companion answered (usually within a second; at most
`[server] command_timeout_secs` + 5 s, plus the time [polite sending](#polite-sending) may hold it, plus up to 12 s of
[busy retries](#busy-retries)).

**Outcomes**

| Status | Body | When |
|---|---|---|
| 202 | `sent: true` | The companion accepted it. |
| 400 | `{"error": {...}}` | Malformed body: no or bad `id`, no `text`, both or neither of `channel` / `to`, `text` with NUL, bad `route` / `path`, `resend_of` with `text` / `to` / `channel`. Nothing sent. |
| 400 | `not_sent`, reason `bad_request` | Empty text, or `route` on a channel message. |
| 400 | `not_sent`, reason `too_long` | Text too long. |
| 404 | `not_sent`, reason `not_found` | Channel not on the companion (also when it was removed before the message's turn came); resend: no such message sent from here, or its channel is no longer on the companion. |
| 404 | `not_sent`, reason `not_contact` | DM target not in the companion's contacts (a DM needs its key, and the other side needs ours to read it). |
| 409 | `not_sent`, reason `ambiguous` | Several contacts match `to`. |
| 409 | `not_sent`, reason `not_dm`, `not_failed`, `expired`, `too_old` | Resend refused (see [Resending](#resending)). |
| 503 | `not_sent`, reason `radio_down` | The companion is not connected. |
| 503 | `not_sent`, reason `busy` | The command queue is full, the companion's packet queue stayed full after the retries, or (channel) the companion's channels are still being read after connecting. |
| 503 | `not_sent`, reason `refused` | The companion said no. |
| 503 | `not_sent`, reason `no_store` | Resend while the message store is off. |
| 503 | `{"error": {"code": "unavailable"}}` | The proxy is stopping. |
| 504 | `unknown` | No answer in time: it may have gone out. Do not resend automatically: ask the user, then send it again with `resend_of` (its `message_id`) and a new `id`. Without a `message_id`, look for the message by its `client_id` first. |

After a send, exactly one `message` event with `"client_id"` and `"outgoing": true` follows (also
when no copy is stored; its `id` is then `null`). A client ties it to its pending send even if the
HTTP answer was lost: that event means it went out or, when the companion did not answer, that it may
have (a `message_status` event with the state `"unknown"` follows). The MeshCore apps on port 5000 get one copy
each when copies are on (see [Storage and copies](#storage-and-copies)).

On the proxy machine: `mcproxy send --channel Public "text"` / `mcproxy send --to "Valley Relay" "text"`.

### DELETE /api/v1/messages

Deletes a whole DM conversation (both ways) from the proxy's store.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `with` | hex string | required | The contact's key or its start (hex, at least 4). A start must fit one contact only. |

```bash
curl -k -X DELETE "https://192.168.1.50:5080/api/v1/messages?with=7c4d2e8f"
```

```json
{ "ok": true, "deleted": 42 }
```

A `messages_deleted` event follows. See [Storage and copies](#storage-and-copies) for what a delete
touches.

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | No `with`, or not hex / shorter than 4. |
| 409 | `ambiguous` | Several contacts' keys start with `with`. Nothing was deleted. |
| 503 | `refused` | The message store is off, or the delete failed. |
| 503 | `unavailable` | The proxy did not answer in time. |

### DELETE /api/v1/messages/{id}

Deletes one stored message.

```bash
curl -k -X DELETE https://192.168.1.50:5080/api/v1/messages/1240
```

```json
{ "ok": true, "deleted": 1 }
```

A `messages_deleted` event follows.

| Status | Code | When |
|---|---|---|
| 400 | (plain text) | `{id}` is not a number. |
| 404 | `not_found` | No such message. |
| 503 | `refused` | The message store is off, or the delete failed. |
| 503 | `unavailable` | The proxy did not answer in time. |

### GET /api/v1/messages/{id}/heard

Every time we heard the packet that carried a message (the first copy and repeats by other
repeaters), oldest first; for our own messages, the repeats heard once matched.

```bash
curl -k https://192.168.1.50:5080/api/v1/messages/1234/heard
```

```json
{
  "matched": true,
  "items": [
    {
      "ts": 1793600400, "snr_db": 6.25, "rssi_dbm": -98, "hops": 2, "route": "flood",
      "path": ["7c", "a1"],
      "via": {
        "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
        "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
        "name": "Hilltop", "kind": "repeater", "lat": 12.3, "lon": 45.6,
        "confidence": 0.92, "choices": 1
      }
    }
  ]
}
```

| Field | Type | Meaning |
|---|---|---|
| `matched` | bool | `false` = no packet was matched to this message (`items` is `[]`). |
| `items[].via` | [hop](#resolved-hops) or null | The repeater that passed it to us; `null` = heard directly from the sender. |

The packet is matched by channel (or sender + us for DMs) and hop count among those heard in the
minute before: [best effort](#deduced-data).

| Status | Code | When |
|---|---|---|
| 404 | `not_found` | No such message. |

## Channels

### GET /api/v1/channels

Every channel known, by id, with its traffic: the companion's slots first (slot order, as last
read), then channels seen on a companion before (`slot: null`, `on_companion: false`; `name: null`
when it is not known).

```bash
curl -k https://192.168.1.50:5080/api/v1/channels
```

```json
{
  "items": [
    { "id": "3f9a1c7e5b2d4f60", "slot": 0, "name": "Public", "kind": "public",
      "on_companion": true, "messages": 812, "last_at": 1793600400, "region": null },
    { "id": "d41e8a2c6b9f0357", "slot": 1, "name": "#hills", "kind": "hashtag",
      "on_companion": true, "messages": 95, "last_at": 1793600500, "region": "eu-se" },
    { "id": "9b0c2e4a6d8f1357", "slot": null, "name": "Hikers", "kind": "private",
      "on_companion": false, "messages": 12, "last_at": 1793000000, "region": null }
  ],
  "next": null
}
```

| Field | Type | Meaning |
|---|---|---|
| `id` | string or null | Channel id, 16 hex (see [Channel ids and slots](#channel-ids-and-slots)). |
| `slot` | integer or null | Slot on the connected companion; `null` = not on it. |
| `name` | string or null | Channel name. |
| `kind` | string | `"public"`, `"hashtag"` or `"private"`. |
| `on_companion` | bool | On the connected companion now. |
| `messages`, `last_at` | integer / integer or null | Messages still stored (all time, within metadata retention) and the newest one's time. For a period, use `GET /api/v1/messages/stats` `by_channel`. |
| `region` | string or null | The channel's own [region](#regions), kept by the proxy: a name, `"*"` (no region), or `null` (the companion's default applies). |

Channel keys are never served.

### POST /api/v1/channels

Adds a channel on the companion, in its first empty slot (a channel already on it is never replaced).

| Field | Type | Required | Meaning |
|---|---|---|---|
| `name` | string | yes | `"#name"` for a hashtag channel, `"Public"`, or any name for a private channel. 1 to 31 bytes, no control characters. |
| `key` | string | private channels | The channel key, 32 hex characters (as the apps share it). |

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/channels \
  -H "Content-Type: application/json" -d '{"name": "#hills"}'
```

```json
{ "ok": true, "slot": 1, "name": "#hills", "id": "d41e8a2c6b9f0357" }
```

Answered once the companion took it; a `channels` event follows.

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | No name, name not 1-31 bytes or with control characters, key not 32 hex. |
| 400 | `needs_key` | A private channel without its key. |
| 409 | `exists` | The companion already has that channel. |
| 409 | `full` | No empty slot. |
| 409 | `busy` | The slots are still being read after connecting; or the message store is off and no app is connected on port 5000 (messages waiting on the companion would be lost). |
| 503 | `radio_down` | The companion is not connected. |
| 503 | `refused` | The companion refused it (or did not answer). |

### PATCH /api/v1/channels/{id}

Sets a channel's own [region](#regions), kept by the proxy for every app (one per channel, the
last one set wins).

`{id}` is the channel id, 16 hex characters, as `GET /api/v1/channels` gives it.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `region` | string or null | yes | A region name, `"*"` (no region), or `null` (the companion's default). |

```bash
curl -k -X PATCH https://192.168.1.50:5080/api/v1/channels/d41e8a2c6b9f0357 \
  -H "Content-Type: application/json" -d '{"region": "eu-se"}'
```

```json
{ "ok": true, "region": "eu-se" }
```

A `channels` event `{"channel_id", "region"}` follows.

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | `{id}` not 16 hex, bad region name. |
| 503 | `no_store` | The message store is off. |
| 503 | `unavailable` | Saving failed, or no answer in time. |

### DELETE /api/v1/channels/{id}

Removes a channel from the companion; its slot is emptied.

`{id}` is the channel's id; else its name; else its slot number. It is removed only if its slot still
holds that channel when the command goes out.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `messages` | string | `keep` | `keep` keeps its stored messages; `delete` also deletes them from the proxy (by channel id), once the companion confirmed the removal. |

```bash
curl -k -X DELETE "https://192.168.1.50:5080/api/v1/channels/d41e8a2c6b9f0357?messages=keep"
```

```json
{ "ok": true, "slot": 1, "name": "#hills", "id": "d41e8a2c6b9f0357" }
```

A `channels` event follows; with `messages=delete`, the messages are deleted once the companion shows
that the channel is gone, and a `channel_messages_deleted` event follows. If the companion does not
show that (its slot still holds the channel, or the link was lost before), the messages are kept and
no such event comes. The channel's region setting is kept: it belongs to the channel's identity and comes back if
the channel is added again.

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | `messages` not `keep` / `delete`. |
| 404 | `not_found` | The companion has no such channel. |
| 409 | `busy` | As for `POST /api/v1/channels`. |
| 503 | `radio_down` | The companion is not connected. |
| 503 | `refused` | The companion refused it (or did not answer). |

## Nodes

### GET /api/v1/nodes

Nodes heard on the mesh and the companion's contacts, most recently heard first.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=0` (everything) | Window on when a node was last heard. |
| `kind` | string | all | `chat`, `repeater`, `room` or `sensor`. |
| `contacts` | string | see below | Which companion contacts outside the window are added: `favorites` or `all`. |
| `mine` | bool | `false` | `true` = only [your own nodes](#your-own-nodes), whatever the window. |
| `limit`, `cursor` | | 500 | Paging (see [Conventions](#conventions)). |

**What is listed, in this order:**

1. Nodes heard in the window, most recently heard first (`in_window: true`).
2. Companion contacts outside the window: **favourites always**, and every contact with
   `contacts=all` or when the window is everything (`hours=0`, the default) (`in_window: false`).
3. With every contact listed: archived contacts that were never heard (`contact: false`,
   `left_companion_at` set). See [Contact archive](#contact-archive-and-other-companions).

With `mine=true`: only your own nodes, most recently heard first, then the ones never heard. Only
the page asked for is built, so smaller pages are cheaper.

```bash
curl -k "https://192.168.1.50:5080/api/v1/nodes?kind=repeater&hours=24&limit=100"
```

```json
{
  "items": [
    {
      "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "name": "Hilltop",
      "kind": "repeater",
      "first_seen_at": 1790000000,
      "last_seen_at": 1793600300,
      "adverts": 412,
      "snr_db": 8.25,
      "rssi_dbm": -88,
      "hops": 0,
      "via": null,
      "lat": 12.3,
      "lon": 45.6,
      "distance_km": 4.2,
      "contact": true,
      "favorite": true,
      "knows_us": true,
      "out_path": [],
      "access": { "role": "admin", "firmware": 1, "at": 1793500000 },
      "mine": true,
      "saved_login": true,
      "path_id_bytes": 1,
      "path_id_bytes_at": 1793600300,
      "left_companion_at": null,
      "other_companion": null,
      "heard": true,
      "in_window": true
    },
    {
      "key": "c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2",
      "name": "Valley Relay",
      "kind": "repeater",
      "first_seen_at": null,
      "last_seen_at": null,
      "adverts": 0,
      "snr_db": null,
      "rssi_dbm": null,
      "hops": null,
      "via": null,
      "lat": null,
      "lon": null,
      "distance_km": null,
      "contact": true,
      "favorite": true,
      "knows_us": null,
      "out_path": ["a1"],
      "access": null,
      "mine": false,
      "saved_login": false,
      "path_id_bytes": null,
      "path_id_bytes_at": null,
      "left_companion_at": null,
      "other_companion": null,
      "heard": false,
      "in_window": false,
      "companion_heard_at": 1793400000
    }
  ],
  "next": null
}
```

**Node object**

| Field | Type | Meaning |
|---|---|---|
| `key` | string | Public key, 64 hex. |
| `name` | string | Node name. |
| `kind` | string | Node kind. |
| `first_seen_at`, `last_seen_at` | integer or null | First and last advert heard; `null` = never heard. |
| `adverts` | integer | Adverts heard. |
| `snr_db`, `rssi_dbm`, `hops` | number / integer or null | From the last advert heard. |
| `via` | [hop](#resolved-hops) or null | The repeater we last heard its advert through, resolved with the whole advert path as context; `null` = direct. |
| `lat`, `lon` | number or null | Position. |
| `distance_km` | number or null | From our companion's position. |
| `contact` | bool | In the companion's contact list. |
| `favorite` | bool or null | Starred on the companion itself (its contact flags, as the MeshCore app shows them); `null` when not a contact. |
| `knows_us` | true or null | `true` when there is proof the node has our key; `null` = unknown. See [Deduced data](#deduced-data). |
| `out_path` | array or null | Contacts: the path the companion uses for DMs to it, repeater ids first hop first; `[]` = direct; `null` = none known (DMs flood) or not a contact. There is no "updated at": the companion does not keep one. |
| `access` | object or null | Our role from the last login answer: `{"role", "firmware", "at"}`; `null` = no login seen. See [Roles and logins](#roles-and-logins). |
| `mine` | bool | One of [your own nodes](#your-own-nodes). |
| `saved_login` | bool | A login password is saved for it. |
| `path_id_bytes`, `path_id_bytes_at` | integer or null | The node's own path hash mode (1-3), from its latest flood advert, and when. See [Path id size](#path-id-size). |
| `left_companion_at` | integer or null | When it left the companion's contact list; `null` = still on it, or never a contact. |
| `other_companion` | string or null | The key of another companion it is a contact of (kept apart); `null` otherwise. |
| `heard` | bool | `false` = a contact the proxy never heard an advert from: values come from its contact frame. |
| `in_window` | bool | `false` = listed because it is a contact / favourite, not heard in the window. |
| `companion_heard_at` | integer or null | **Only on items with `heard: false`**: when the companion last updated the contact. |

Contacts follow the companion (the proxy's contact cache). Names and keys of archived contacts are
used everywhere a key is resolved (DM senders, `{key}` prefixes).

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | Unknown `kind`; `contacts` not `all` / `favorites`. |

### GET /api/v1/nodes/{key}

One node: the [node object](#get-apiv1nodes) (also for a contact never heard, or an archived
contact), plus its latest answers and its resolved DM path.

```bash
curl -k https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4
```

```json
{
  "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "name": "Hilltop",
  "kind": "repeater",
  "first_seen_at": 1790000000,
  "last_seen_at": 1793600300,
  "adverts": 412,
  "snr_db": 8.25,
  "rssi_dbm": -88,
  "hops": 0,
  "via": null,
  "lat": 12.3,
  "lon": 45.6,
  "distance_km": 4.2,
  "contact": true,
  "favorite": true,
  "knows_us": true,
  "out_path": [],
  "access": { "role": "admin", "firmware": 1, "at": 1793500000 },
  "mine": true,
  "saved_login": true,
  "path_id_bytes": 1,
  "path_id_bytes_at": 1793600300,
  "left_companion_at": null,
  "other_companion": null,
  "tracked": true,
  "latest": {
    "status": { "at": 1793590000, "battery_mv": 4100, "uptime_secs": 86400, "noise_floor": -110 },
    "owner": { "at": 1793000000, "firmware": "v1.9.1", "owner": "Mountain club" },
    "neighbours": { "at": 1793500000, "total": 12 }
  },
  "out_path_nodes": []
}
```

| Field | Type | Meaning |
|---|---|---|
| `tracked` | bool | Asked for status + telemetry regularly. |
| `latest.status` | object or null | Latest status answer: `at`, `battery_mv`, `uptime_secs`, `noise_floor` (each may be `null`). `null` = never answered. |
| `latest.owner` | object or null | Latest owner answer: `at`, `firmware`, `owner`. |
| `latest.neighbours` | object or null | Latest neighbour table: `at`, `total`. |
| `out_path_nodes` | array or null | `out_path` [resolved](#resolved-hops), same order (first hop first), read as the way back from the contact to us so each id has its neighbours as context. `null` when `out_path` is `null`; `[]` = direct. |

`heard` and `in_window` appear here only for a contact never heard (both `false`, with
`companion_heard_at`); a heard node has neither.

| Status | Code | When |
|---|---|---|
| 404 | `not_found` | Unknown node. |

### PATCH /api/v1/nodes/{key}

Changes a node: its favourite star and DM path on the companion, and tracking and "mine" on the
proxy. Any subset of the fields:

| Field | Type | Meaning |
|---|---|---|
| `favorite` | bool | Stars / unstars the contact **on the companion** (written back with only its favourite flag changed, like the MeshCore app does). The MeshCore apps see it the next time they read the contacts. |
| `track` | bool | Ask this repeater / room / sensor for its status and telemetry every `[observer] track_interval_mins` (default 720, every 12 hours; one LoRa request each), or stop. Logs in as guest (with the node's saved password, else `[observer] track_password`) when it does not answer. Saved in `[observer] track`, like `mcproxy track add / remove`. Results: `GET /api/v1/nodes/{key}/status` and `/telemetry`. |
| `mine` | bool or null | `true` adds it to [your own nodes](#your-own-nodes), `false` keeps it off the list even with an admin login, `null` goes back to automatic. |
| `path` | array or null | The path the companion uses for DMs to this contact: `null` = forget it (the next DM floods), `[]` = direct, `["a1", "b2"]` = through these repeaters, first hop first (1-3 bytes each, all the same size). Stored on the companion (the MeshCore apps see it). |

```bash
curl -k -X PATCH https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4 \
  -H "Content-Type: application/json" -d '{"favorite": true, "track": true}'
```

```json
{
  "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "track": true,
  "favorite": true
}
```

The answer holds `key` and each field changed, with the resulting value (`mine` as it is now,
`out_path` for `path`). The fields are applied one by one (`track`, `mine`, `favorite`, `path`):
when one fails, the ones before it stay changed. A `node` event follows a change of `favorite`,
`mine` or `path`.

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | Another field; `favorite` / `track` not a bool; `mine` not bool / null; bad `path`; `track: true` on a chat node (only repeaters, rooms and sensors answer). |
| 404 | `not_contact` | `favorite`, `path` or `track: true` on a node that is not a contact of the companion. |
| 503 | `no_store` | `mine` while the message store is off. |
| 503 | `unavailable` | The companion is not connected, refused, or did not answer. |

### GET /api/v1/nodes/{key}/adverts

Every advert heard from a node, oldest first.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Time range. |
| `limit`, `cursor` | | 500 | Paging. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/nodes/c3d4e5f6/adverts?hours=48"
```

```json
{
  "items": [
    {
      "ts": 1793590000, "snr_db": 5.5, "rssi_dbm": -97, "hops": 1, "route": "flood",
      "path": ["a1"],
      "path_nodes": [
        { "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
          "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
          "name": "Hilltop", "kind": "repeater", "lat": 12.3, "lon": 45.6,
          "confidence": 0.95, "choices": 1 }
      ],
      "via": { "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "...": "..." },
      "advert_ts": 1793589990,
      "repeat": false
    }
  ],
  "next": null
}
```

| Field | Type | Meaning |
|---|---|---|
| `route` | string | `"flood"` / `"flood+transport"` = a flood advert; `"direct"` / `"direct+transport"` = a zero-hop advert (sent direct, path length 0, heard only by nodes in range). |
| `path` | array | Path ids as received. |
| `path_nodes` | array | Every id of `path` [resolved](#resolved-hops), same order, with this advert's path and sender as context; `[]` = heard directly. |
| `via` | hop or null | The last repeater before us; `null` = direct. |
| `advert_ts` | integer or null | The sender's clock from the advert payload (one value per distinct advert). |
| `repeat` | bool | The same advert heard again by another path (same `advert_ts` as the row before it). |

### GET /api/v1/nodes/{key}/heard

How we hear this node over time, also without adverts: every packet it relayed to us (it is the
last hop of a flood path) plus its own adverts heard directly.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Time range. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/heard?hours=2"
```

```json
{
  "packets": 230,
  "avg_snr_db": 7.4,
  "avg_rssi_dbm": -90.2,
  "same_hash_nodes": 1,
  "per_hour": [
    { "ts": 1793595600, "packets": 120, "avg_snr_db": 7.6, "avg_rssi_dbm": -89.8 },
    { "ts": 1793599200, "packets": 110, "avg_snr_db": 7.2, "avg_rssi_dbm": -90.6 }
  ],
  "by_type": [
    { "type": "advert", "packets": 90 },
    { "type": "channel", "packets": 80 }
  ]
}
```

| Field | Type | Meaning |
|---|---|---|
| `per_hour` | array | [Dense](#dense-hourly-series), one row per hour. |
| `same_hash_nodes` | integer | Other known nodes whose key starts with the same byte. A last hop is only a short id, so their relays may be counted here too: show a hint when > 0. |

Averages are rounded to one decimal.

### GET /api/v1/nodes/{key}/status

Status answers of a repeater or room server, oldest first.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Time range. |
| `limit`, `cursor` | | 500 | Paging. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/status?hours=168"
```

```json
{
  "items": [
    {
      "ts": 1793590000, "battery_mv": 4100, "noise_dbm": -110, "rssi_dbm": -92, "snr_db": 6.5,
      "tx_queue": 0, "recv": 51200, "sent": 20480,
      "recv_flood": 48000, "recv_direct": 3200, "sent_flood": 19000, "sent_direct": 1480,
      "airtime_tx_secs": 3600, "airtime_rx_secs": 7200,
      "dups_flood": 9000, "dups_direct": 120, "errors": 0, "recv_errors": 14, "uptime_secs": 864000
    }
  ],
  "next": null
}
```

| Field | Type | Meaning |
|---|---|---|
| `noise_dbm` | integer or null | The node's noise floor. |
| `rssi_dbm`, `snr_db` | | Of the last packet the node received. |
| `tx_queue` | integer or null | Packets in its send queue. |
| `recv`, `sent`, `recv_flood`, `recv_direct`, `sent_flood`, `sent_direct` | integer or null | Packet counters. |
| `airtime_tx_secs`, `airtime_rx_secs` | integer or null | Air time. |
| `dups_flood`, `dups_direct` | integer or null | Duplicates dropped. |
| `errors` | integer or null | Error events. |
| `recv_errors` | integer or null | Receive errors. |
| `uptime_secs` | integer or null | Since the node booted. |

Every value can be `null` (not every firmware reports it).

### GET /api/v1/nodes/{key}/telemetry

Every telemetry value of a node, oldest first, one row per value.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Time range. |
| `limit`, `cursor` | | 500 | Paging. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/telemetry"
```

```json
{
  "items": [
    { "ts": 1793590000, "channel": 1, "kind": "voltage", "value": 4.1 },
    { "ts": 1793590000, "channel": 1, "kind": "temperature", "value": 6.5 },
    { "ts": 1793590000, "channel": 2, "kind": "humidity", "value": 71.0 }
  ],
  "next": null
}
```

| Field | Type | Meaning |
|---|---|---|
| `channel` | integer | The sensor channel (Cayenne LPP). |
| `kind` | string | `"voltage"`, `"temperature"`, `"humidity"`, ... (the LPP type). |
| `value` | number | The value, in the type's usual unit. |

## Mesh and statistics

### GET /api/v1/mesh/summary

Totals of the packets heard, for charts.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Time range. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/mesh/summary?hours=24"
```

```json
{
  "packets": 5000,
  "avg_snr_db": 6.5,
  "avg_rssi_dbm": -95.0,
  "by_type": [
    { "type": "advert", "packets": 900, "avg_snr_db": 5.0 },
    { "type": "channel", "packets": 700, "avg_snr_db": 6.8 }
  ],
  "per_hour": [
    { "ts": 1793595600, "packets": 200, "messages": 35, "avg_snr_db": 6.0,
      "by_type": { "advert": 60, "dm": 5, "channel": 30, "path": 40, "ack": 65 } },
    { "ts": 1793599200, "packets": 0, "messages": 0, "avg_snr_db": null, "by_type": {} }
  ],
  "via": [
    { "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "name": "Hilltop", "kind": "repeater", "lat": 12.3, "lon": 45.6,
      "confidence": 0.98, "choices": 1, "packets": 800, "avg_snr_db": 7.0, "id_bytes": 1 }
  ]
}
```

| Field | Type | Meaning |
|---|---|---|
| `avg_snr_db`, `avg_rssi_dbm` | number or null | Plain averages over the packets (see [SNR and RSSI](#snr-and-rssi)). |
| `by_type` | array | Packets per type (`type` `null` = not decoded), most first. Types: `request`, `response`, `dm`, `ack`, `advert`, `channel`, `channel-data`, `anon-request`, `path`, `trace`, `multipart`, `control`, `raw`, `other`. |
| `per_hour` | array | [Dense](#dense-hourly-series). |
| `per_hour[].by_type` | object | That hour's packets per type (same names; `"unknown"` for a type not decoded). A type not heard that hour is left out (read it as 0). The values add up to `packets`. |
| `per_hour[].messages` | integer | `dm` + `channel` packets heard that hour (every packet heard, not only the ones for us; repeats count once per time heard). The message share of the traffic is `messages / packets`. |
| `via` | array | The **last id of flood packets' paths**: the repeater our companion heard the packet from. A [resolved hop](#resolved-hops) plus `packets`, `avg_snr_db` and `id_bytes` (1-3). See below. |

**Reading `via`.** Direct-routed packets are not counted (their path lists the hops still to come).
There is one entry per id string, so it counts ids, not repeaters: the same repeater appears once
per id size the mesh uses (`id_bytes`), one 1-byte id can stand for several repeaters (resolved as
the last hop before us: `confidence`, `could_be`), and ids no heard repeater has (`choices: 0`) are
often other meshes or damaged packets that passed the radio's check. For "repeaters heard
directly", count entries with `choices` > 0 and a few packets or more, or use
`GET /api/v1/mesh/graph` (links into `"self"`).

### GET /api/v1/mesh/packets

Every packet heard, decoded, oldest first, paged by packet id.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `since_id` | integer | 0 | Packets with an id above this one. |
| `limit` | integer | 1000 | 1 to 5000. |

`next` = the last `id` of the page (a string), or `null` when the page is not full: pass it back
as `since_id`. This endpoint takes no time range.

```bash
curl -k "https://192.168.1.50:5080/api/v1/mesh/packets?since_id=0&limit=2"
```

```json
{
  "items": [
    {
      "id": 98001, "ts": 1793600400, "snr_db": 6.25, "rssi_dbm": -98,
      "route": "flood", "type": "channel", "version": 0, "hops": 2, "hash_size": 1,
      "path": ["7c", "a1"],
      "path_nodes": [],
      "last_hop": "a1", "size_bytes": 64,
      "node_key": null, "dst": null, "src": null, "channel_hash": "3f", "ack": null,
      "scoped": false, "region": null
    },
    {
      "id": 98002, "ts": 1793600402, "snr_db": 8.0, "rssi_dbm": -90,
      "route": "flood", "type": "advert", "version": 0, "hops": 0, "hash_size": 1,
      "path": [], "path_nodes": [],
      "last_hop": null, "size_bytes": 110,
      "node_key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "dst": null, "src": null, "channel_hash": null, "ack": null,
      "scoped": true, "region": "eu"
    }
  ],
  "next": "98002"
}
```

| Field | Type | Meaning |
|---|---|---|
| `id`, `ts`, `rssi_dbm`, `version`, `hops`, `size_bytes` | integer | Packet id, time, RSSI, payload version, hop count, size. |
| `snr_db` | number | SNR. |
| `route` | string | `"flood"`, `"flood+transport"`, `"direct"`, `"direct+transport"`. |
| `type` | string | Payload type (see `mesh/summary` `by_type`). |
| `hash_size` | integer | Path id size of this packet, 1-3. |
| `path` | array | Path ids, lowercase hex. |
| `path_nodes` | array | Every id of `path` [resolved](#resolved-hops), same order. Flood packets: the way it came (origin, repeaters, us), each id resolved with its neighbours and the sender (`node_key`, adverts) as context. Direct packets: their path lists the hops still ahead, so each id is resolved on its own. `[]` = no path. |
| `last_hop` | string or null | Last id of the path. |
| `node_key` | string or null | Adverts: the sender's key. |
| `dst`, `src` | string or null | Destination / source hashes (DMs, requests). |
| `channel_hash` | string or null | Channel hash (the first byte of the channel id). |
| `ack` | string or null | ACK code. |
| `scoped` | bool | Sent in a region (flood scope). |
| `region` | string or null | That region's name, when it matched a known one. |

### GET /api/v1/mesh/graph

The mesh as a graph built over time from everything heard: every packet path (two nodes next to
each other in a path hear each other), the repeaters' neighbour tables and traces.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since` | integer | `hours=720` (30 days) | Window; `hours=0` = everything. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/mesh/graph?hours=720"
```

```json
{
  "nodes": [
    { "id": "self", "key": null, "name": "Base Station", "kind": "self", "lat": 12.31, "lon": 45.62,
      "hash": null, "confidence": 1.0, "choices": 1, "packets": 0, "last_seen": null },
    { "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "name": "Hilltop", "kind": "repeater", "lat": 12.3, "lon": 45.6,
      "hash": "a1", "confidence": 0.92, "choices": 1, "packets": 1200, "last_seen": 1793600400 },
    { "id": "h:5d", "key": null, "name": null, "kind": "unknown", "lat": null, "lon": null,
      "hash": "5d", "confidence": 0.0, "choices": 0, "packets": 14, "last_seen": 1793500000 }
  ],
  "links": [
    { "from": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "to": "self",
      "heard_forward": 800, "heard_back": 0, "snr_forward_db": 7.0, "snr_back_db": null,
      "last_seen": 1793600400, "sources": ["paths"] }
  ],
  "shared_ids": [
    { "id": "7c", "nodes": [
      { "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f", "name": "Valley Relay",
        "last_seen": 1793600000, "distance_km": 9.8 },
      { "key": "7c11aa22bb33cc44dd55ee66ff7788997c11aa22bb33cc44dd55ee66ff778899", "name": "Ridge Repeater",
        "last_seen": 1793400000, "distance_km": 31.5 }
    ] }
  ],
  "repeaters": [
    { "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "name": "Hilltop", "kind": "repeater", "path_id_bytes": 1 }
  ]
}
```

See [Mesh graph objects](#mesh-graph-objects) for `nodes` and `links`.

| Field | Type | Meaning |
|---|---|---|
| `shared_ids` | array | Path ids matching more than one repeater we heard: on a direct route both may forward the packet (the firmware forwards on an id match), and maps cannot be sure which one a path meant. |
| `repeaters` | array | Every repeater and room heard (all time, not only the window), with its own path hash mode (`path_id_bytes` 1-3 or `null`), for route building on the client. |

### GET /api/v1/mesh/delivery

`GET /api/v1/mesh/delivery?hours=720[&node=<key>]`: how the DMs we sent fared, from any app,
counted per contact and per route the companion used. Every attempt counts, retries included, under
the route that attempt really took. Raw counts: weighing routes (per link, against signal) is up to
the client. `node` = only the contacts whose key starts with it (hex). Default 30 days, `hours=0` =
everything; at most 2000 rows, most attempts first.

```json
{ "paths": [
    { "to": "<contact key>", "name": "Alice", "path": ["a1", "7c"],
      "sent": 12, "delivered": 10, "failed": 2, "rtt_ms_median": 2650,
      "last_at": 1793600103, "last_delivered_at": 1793600103 }
  ],
  "since": 1791008103 }
```

| Field | Type | Meaning |
|---|---|---|
| `to` | string | The contact's public key; the 12-hex key start when it is not a contact of the companion. |
| `name` | string or null | Its name, when heard advertising. |
| `path` | array or null | The route: `null` = flooded, `[]` = direct (no repeater), else the path ids (as in the packet: 1-3 bytes hex), first hop first. It is the path the companion held for the contact when the attempt went out; an attempt whose path is not known is not counted. |
| `sent` | integer | Attempts by this route. |
| `delivered`, `failed` | integer | Attempts whose ACK came back (a late ACK too) / that got none in time. The rest are still waiting. |
| `rtt_ms_median` | integer or null | Median ACK round trip of the delivered ones. |
| `last_at`, `last_delivered_at` | integer or null | The latest attempt by this route, and the latest one delivered. |

Kept as long as the node history (`[observer] node_history_days`).

### GET /api/v1/mesh/hours

Traffic by hour of the day (the proxy machine's local time).

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since` | integer | `hours=720` | Window; `hours=0` = everything. |
| `node` | string | none | A node key or prefix: only what we heard from that node (its own adverts heard directly, and what it relayed to us). |

With `node`: the node's daily pattern (a solar repeater going quiet at night, the best hours to
reach it). Without: all traffic, plus our noise floor by hour. Best / silent hours are the client's
reading of `hours`.

```bash
curl -k "https://192.168.1.50:5080/api/v1/mesh/hours?hours=720&node=a1b2c3d4"
```

```json
{
  "days": 30.0,
  "hours": [
    { "hour": 0, "packets": 310, "packets_per_day": 10.3, "snr_median_db": 6.5, "snr_low_db": 2.0,
      "snr_high_db": 9.5, "rssi_avg": -91.2, "noise_floor_avg": null }
  ],
  "week": [
    [12, 9, 8, 7, 6, 8, 14, 20, 25, 27, 26, 28, 30, 29, 27, 26, 25, 24, 22, 20, 18, 16, 14, 13]
  ]
}
```

| Field | Type | Meaning |
|---|---|---|
| `days` | number | Length of the window in days (at least 1). |
| `hours` | array | 24 rows, `hour` 0-23. |
| `hours[].snr_low_db`, `snr_high_db` | number or null | 10th / 90th percentile SNR. |
| `hours[].rssi_avg` | number or null | Average RSSI (dBm). |
| `hours[].noise_floor_avg` | number or null | Our noise floor (dBm); only without `node`. |
| `week` | array | 7 rows (Monday first) of 24 packet counts (shortened in the example). |

| Status | Code | When |
|---|---|---|
| 400 / 404 / 409 | | Bad, unknown or ambiguous `node`. |

### GET /api/v1/mesh/replay

Every packet in a window with the nodes it went through, for an animated map.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `until` | integer | now | End of the window. |
| `since` | integer | `until` - `hours` | Start of the window. |
| `hours` | integer | 1 | Length of the window when `since` is not given (at least 1). |
| `limit` | integer | 5000 | 1 to 20000 packets. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/mesh/replay?hours=1"
```

```json
{
  "since": 1793596800,
  "until": 1793600400,
  "nodes": {
    "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90":
      { "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
        "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
        "name": "Hilltop", "kind": "repeater", "lat": 12.3, "lon": 45.6 },
    "self": { "id": "self", "key": null, "name": "Base Station", "kind": "self", "lat": 12.31, "lon": 45.62 }
  },
  "packets": [
    { "id": 98001, "ts": 1793600400, "type": "channel", "route": "flood", "snr": 6.25,
      "chain": ["h:7c", "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "self"],
      "confidence": [0.0, 0.92, 1.0] }
  ],
  "more": false
}
```

| Field | Type | Meaning |
|---|---|---|
| `nodes` | object | Each node of the chains once, by id: `id`, `key`, `name`, `kind`, `lat`, `lon` (no statistics). |
| `packets[].chain` | array of ids | Origin (when known), repeaters, then `"self"`. A path is followed only for flood packets (not for direct packets or traces). |
| `packets[].confidence` | array of numbers | How sure each `chain[i]` is (origin and us: 1). See [Resolved hops](#resolved-hops). |
| `more` | bool | The window held more than `limit` packets: ask again from the last `ts`. |

### GET /api/v1/mesh/adverts

The raw advert times of every node heard advertising in the window, for the client to find timers
(an advert does not carry its interval) and adverts sent by hand.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since` | integer | `hours=720` | Window on when we first heard each advert; `hours=0` = everything. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/mesh/adverts?hours=720"
```

```json
{
  "nodes": [
    {
      "node": { "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
                "name": "Hilltop", "kind": "repeater" },
      "flood_ts": [1793514000, 1793557200, 1793600400],
      "local_ts": [1793580000],
      "last_heard_at": 1793600410
    }
  ],
  "since": 1791008400
}
```

| Field | Type | Meaning |
|---|---|---|
| `flood_ts` | array of integers | Flood adverts: the sender's clock from the advert itself (exact, one value per distinct advert, whatever the delay on the way), oldest first, the latest 500. |
| `local_ts` | array of integers | Zero-hop adverts (sent direct, heard only in range), the same way. |
| `last_heard_at` | integer | When we last heard one of its adverts. |
| `since` | integer | Start of the window used. |

### GET /api/v1/messages/stats

Message counts for charts.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Time range. |
| `channel` | string | all | A channel id, else a name, else a slot number (DMs are then left out). |
| `from` | string | all | A sender name (any case) or key prefix: received messages only. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/messages/stats?hours=48"
```

```json
{
  "per_hour": [ { "ts": 1793595600, "dm": 2, "channel": 14 } ],
  "per_day": [ { "ts": 1793577600, "dm": 9, "channel": 120 } ],
  "by_channel": [
    { "channel": "Public", "channel_id": "3f9a1c7e5b2d4f60", "messages": 90 },
    { "channel": "#hills", "channel_id": "d41e8a2c6b9f0357", "messages": 30 }
  ],
  "by_sender": [
    { "name": "Hilltop", "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "messages": 25 },
    { "name": "Trail Walker", "key": null, "messages": 11 }
  ]
}
```

| Field | Type | Meaning |
|---|---|---|
| `per_hour` | array | [Dense](#dense-hourly-series), `ts` = start of the hour. |
| `per_day` | array | Dense the same way, `ts` = midnight UTC. |
| `by_channel` | array | Per channel, most first (`channel` is `""` when its name is not known; `channel_id` is `null` for messages whose channel cannot be told). |
| `by_sender` | array | Per sender, most first (`key` `null` when not known exactly). DMs we sent are not counted; our channel messages count under our node name (`key` `null`). |

| Status | Code | When |
|---|---|---|
| 404 | `not_found` | `channel` matches no channel. |

### GET /api/v1/messages/senders

Everyone who wrote, most recent first. Messages we sent are not counted.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=0` (everything) | Time range. |
| `limit`, `cursor` | | 500 | Paging. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/messages/senders"
```

```json
{
  "items": [
    { "name": "Hilltop", "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "messages": 40, "dm": 4, "channel": 36, "channels": ["#hills", "Public"],
      "first_at": 1790000000, "last_at": 1793600400 }
  ],
  "next": null
}
```

| Field | Type | Meaning |
|---|---|---|
| `key` | string or null | DM senders: the contact's key; channel senders: the key when exactly one node heard has that name. |
| `channels` | array of strings | Channels it wrote in (names, or ids when the name is not known). |

## Node actions

Repeaters, room servers and sensors have a CLI, not a chat. Our role on a node decides what we may
do with it: see [Roles and logins](#roles-and-logins). Every action below goes over LoRa and can take
from seconds to a minute or more: show a waiting state.

### POST /api/v1/nodes/{key}/login

Logs in to a node. With no answer the login is tried up to 3 times, the last one flooded (the
companion forgets the route, as for a DM's last try), so a failure can take a minute or more.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `password` | string | no | At most 32 bytes. Blank works when the node has us on its access list, or has a blank guest password (the default on a repeater; a room server's default is usually `hello`). On a room server, the guest password gives `read_write`. |
| `save` | bool | no | `true` keeps this password once the login worked (needs a password); `false` forgets the saved one once the login worked; absent = no change. |
| `use_saved` | bool | no | `true` logs in with the saved password; goes without `password` and `save`. |

The body is optional (no body = a blank login).

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/login \
  -H "Content-Type: application/json" -d '{"use_saved": true}'
```

```json
{ "ok": true, "role": "admin", "firmware": 1, "saved_login": true }
```

| Field | Type | Meaning |
|---|---|---|
| `role` | string or null | `"admin"`, `"read_write"`, `"read_only"`, `"guest"`; `null` = an older node that does not say. |
| `firmware` | integer or null | The node's firmware level from the login answer. |
| `saved_login` | bool | A password is saved for it now. |

A `node` event follows. The password goes to the node and is not written to the log.

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | Password over 32 bytes; `save: true` without a password; `use_saved` with `password` or `save`. |
| 403 | `refused` | The node refused the login. |
| 404 | `not_contact` | Not a contact of the companion. |
| 409 | `no_saved` | `use_saved` with no saved password. |
| 409 | `busy` | A login to this node is under way. |
| 503 | `radio_down` | The companion is not connected, or was lost during the login. |
| 503 | `no_store` | The message store is off. |
| 503 | `unavailable` | The proxy is stopping. |
| 504 | `no_answer` | Out of range, or a wrong password: a repeater never answers one, and neither does a room server with `allow.read.only` off, so the two cannot be told apart. A blank login on a repeater with a guest password set ends the same way, unless it already has us on its access list. |

### DELETE /api/v1/nodes/{key}/login

Forgets the node's saved password.

```bash
curl -k -X DELETE https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/login
```

```json
{ "ok": true }
```

A `node` event follows.

| Status | Code | When |
|---|---|---|
| 503 | `no_store` | The message store is off. |
| 503 | `unavailable` | Saving failed, or the proxy is stopping. |
| 504 | `no_answer` | No answer from the proxy in time. |

### POST /api/v1/nodes/{key}/cli

Sends one CLI command to a node, as meshcore-cli does (the CLI message type, once), and answers with
its reply. Commands go one at a time per node (the others wait: a reply does not say which command
it answers). A reply comes in seconds, or tens of seconds over several hops; polite sending may
delay it further.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `command` | string | yes | 1 to 160 bytes, one line (no control characters). |

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/cli \
  -H "Content-Type: application/json" -d '{"command": "ver"}'
```

```json
{ "ok": true, "id": 57, "reply": "v1.9.1 (Build: 01-Jan-2026)" }
```

A `cli` event follows each command. **Never send a command again by yourself:** the companion gives
each one a new timestamp, so the node runs it again.

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | No command, not 1 to 160 bytes, or control characters: not sent. |
| 403 | `not_admin` | Our last login gave another role: not sent (the node would ignore it). |
| 403 | `refused` | The companion did not take it: not sent. |
| 404 | `not_found` / 409 `ambiguous` | The key matches no node, or several: not sent. |
| 404 | `not_contact` | Not a contact of the companion: not sent. |
| 409 | `not_supported` | A chat node: not sent. |
| 409 | `busy` | Too many commands are waiting (32): not sent. |
| 503 | `radio_down` | The companion is not connected: not sent. |
| 503 | `no_store` | The message store is off (commands need it for their history): not sent. |
| 503 | `unavailable` | The proxy is stopping: not sent. |
| 504 | `no_answer` | No reply in time: out of range, or not an admin. **It may have run.** |
| 504 | `unknown` | The companion disconnected while the command was waiting or under way. **It may have run.** |

### GET /api/v1/nodes/{key}/cli

The commands sent from here to a node and their replies, newest first. Every device sees this list.
Kept encrypted, as long as message texts.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `limit` | integer | 200 | 1 to 2000. |
| `cursor` | string | none | Continue below this command id (from `next`). |

```bash
curl -k https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/cli
```

```json
{
  "items": [
    { "id": 58, "ts": 1793600600, "state": "ok", "command": "password ***", "reply": "***" },
    { "id": 57, "ts": 1793600500, "state": "ok", "command": "ver", "reply": "v1.9.1 (Build: 01-Jan-2026)" }
  ],
  "next": null
}
```

| Field | Type | Meaning |
|---|---|---|
| `state` | string | `"ok"`, `"no_answer"`, `"refused"` or `"unknown"`. |
| `command`, `reply` | string or null | Commands with a secret (`password`, `set guest.password`, ...) show as `password ***` with the reply `***`; only the `POST` that sent one gets its real reply. `null` when the text cannot be read. |

### GET /api/v1/cli/commands

The CLI commands MeshCore nodes have, in display order, for a list to pick from.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `kind` | string | all | `repeater`, `room` or `sensor`: only the commands this kind of node has. |
| `serial` | bool | `false` | Include commands that work only on the node's own USB console. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/cli/commands?kind=repeater"
```

```json
{
  "items": [
    { "usage": "set tx <dbm>", "about": "Set the transmit power (1-22 dBm)", "section": "Radio",
      "only": null, "needs": null, "danger": true, "secret": false, "serial": false },
    { "usage": "get radio.rxgain", "about": "RX boosted gain state", "section": "Radio",
      "only": null, "needs": "firmware 1.14.1", "danger": false, "secret": false, "serial": false }
  ]
}
```

| Field | Type | Meaning |
|---|---|---|
| `only` | array or null | The node kinds that have it; `null` = all. |
| `needs` | string or null | Not every node has it (a firmware version, supported boards). |
| `danger` | bool | Ask before sending: it restarts the node, cuts it off or changes how it transmits. |
| `secret` | bool | It carries a secret (masked in the history). |
| `serial` | bool | Only on the node's own USB console (left out unless `serial=true`). |

### GET /api/v1/nodes/{key}/neighbours

The neighbour table a repeater gave when last asked: the nodes **it** hears directly, strongest first.

```bash
curl -k https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/neighbours
```

```json
{
  "asked_at": 1793500000,
  "total": 12,
  "items": [
    { "prefix": "7c4d2e8f", "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
      "name": "Valley Relay", "kind": "repeater", "us": false,
      "heard_at": 1793499400, "heard_secs_ago": 600, "snr_db": 9.25 },
    { "prefix": "e5f60718", "key": "e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4",
      "name": "Base Station", "kind": "chat", "us": true,
      "heard_at": 1793499880, "heard_secs_ago": 120, "snr_db": 6.0 }
  ]
}
```

| Field | Type | Meaning |
|---|---|---|
| `asked_at` | integer or null | When the table was read; `null` = never (then `total` is `null` and `items` `[]`). |
| `total` | integer or null | How many neighbours it has (at most 88 are read). |
| `items[].key`, `name`, `kind` | string or null | Resolved from the nodes and contacts we know; `null` if unknown. |
| `items[].us` | bool | It is our companion. |
| `items[].heard_at` | integer | When the repeater last heard that one. |

### POST /api/v1/nodes/{key}/neighbours

Asks the repeater for its neighbour table now and answers like `GET`. LoRa requests: one per 11
entries, plus a guest login when it does not know us; can take a minute. Asking the same node again
while a request is on its way waits for the same answer (no second packet).

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/neighbours
```

The answer has the same shape as [`GET /api/v1/nodes/{key}/neighbours`](#get-apiv1nodeskeyneighbours).
A `node` event follows.

| Status | Code | When |
|---|---|---|
| 404 | `not_contact` | Not a contact of the companion. |
| 503 | `unavailable` | The companion is not connected, the message store is off, or the proxy is stopping. |
| 504 | `no_answer` | Out of range, or the login was refused (a password can be saved with the login endpoint). |

### GET /api/v1/nodes/{key}/owner

The firmware version, name and owner text (set by its admin) a node gave when last asked.

```bash
curl -k https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/owner
```

```json
{ "asked_at": 1793000000, "firmware": "v1.9.1", "name": "Hilltop", "owner": "Mountain club" }
```

All fields are `null` when it was never asked.

### POST /api/v1/nodes/{key}/owner

Asks the node now and answers like `GET`. Repeaters answer; room servers answer depending on their
firmware. Asking again while a request is on its way waits for the same answer.

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/owner
```

The answer has the same shape as [`GET /api/v1/nodes/{key}/owner`](#get-apiv1nodeskeyowner). A
`node` event follows.

| Status | Code | When |
|---|---|---|
| 404 | `not_contact` | Not a contact of the companion. |
| 503 | `unavailable` | The companion is not connected, the message store is off, or the proxy is stopping. |
| 504 | `no_answer` | Out of range, or the login was refused. |

### POST /api/v1/nodes/{key}/status

Asks the node for its status now (one LoRa request, plus a guest login when it does not know us; can
take a minute) and answers like [`GET /api/v1/nodes/{key}/status`](#get-apiv1nodeskeystatus), with
this answer only.

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/status
```

```json
{
  "items": [
    { "ts": 1793600700, "battery_mv": 4100, "noise_dbm": -110, "rssi_dbm": -92, "snr_db": 6.5,
      "tx_queue": 0, "recv": 51300, "sent": 20500, "recv_flood": 48100, "recv_direct": 3200,
      "sent_flood": 19020, "sent_direct": 1480, "airtime_tx_secs": 3605, "airtime_rx_secs": 7210,
      "dups_flood": 9010, "dups_direct": 120, "errors": 0, "recv_errors": 14, "uptime_secs": 864700 }
  ],
  "next": null
}
```

A `node` event follows.

| Status | Code | When |
|---|---|---|
| 404 | `not_contact` | Not a contact of the companion. |
| 503 | `unavailable` | The companion is not connected, the message store is off, or the proxy is stopping. |
| 504 | `no_answer` | Out of range, or the login was refused. |

### POST /api/v1/nodes/{key}/telemetry

Asks the node for its telemetry now and answers like
[`GET /api/v1/nodes/{key}/telemetry`](#get-apiv1nodeskeytelemetry), with this answer only. A guest
gets base telemetry only: battery and MCU temperature.

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/telemetry
```

```json
{
  "items": [
    { "ts": 1793600700, "channel": 1, "kind": "voltage", "value": 4.1 },
    { "ts": 1793600700, "channel": 1, "kind": "temperature", "value": 6.5 }
  ],
  "next": null
}
```

A `node` event follows. Errors as for `POST /api/v1/nodes/{key}/status`.

### POST /api/v1/trace

Sends a trace along a list of repeaters (direct, one LoRa packet each hop); each one notes the SNR
it heard it at.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `path` | array of hex strings | yes | At least one repeater id, all the same size: 1 or 2 bytes each. |
| `round_trip` | bool | no (default `true`) | Add the way back (`a1 > b2 > a1`), so the answer comes back to us. |

For a repeater contact, use its `out_path` plus its own id; for a chat node, its `out_path` (it does
not repeat). The answer comes once the trace is back (up to about a minute; the request waits at most
120 s).

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/trace \
  -H "Content-Type: application/json" -d '{"path": ["a1", "7c"], "round_trip": true}'
```

```json
{
  "path": ["a1", "7c", "a1"],
  "hops": [
    { "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "name": "Hilltop",
      "kind": "repeater", "lat": 12.3, "lon": 45.6, "confidence": 0.95, "choices": 1, "snr_db": 8.5 },
    { "hash": "7c", "id": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
      "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f", "name": "Valley Relay",
      "kind": "repeater", "lat": 12.4, "lon": 45.7, "confidence": 0.8, "choices": 2,
      "could_be": [
        { "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
          "name": "Valley Relay", "last_seen": 1793600000, "distance_km": 9.8 },
        { "key": "7c11aa22bb33cc44dd55ee66ff7788997c11aa22bb33cc44dd55ee66ff778899",
          "name": "Ridge Repeater", "last_seen": 1793400000, "distance_km": 31.5 }
      ],
      "snr_db": 4.25 },
    { "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "...": "...", "snr_db": 5.0 }
  ],
  "final_snr_db": 7.75
}
```

| Field | Type | Meaning |
|---|---|---|
| `path` | array | The path sent (with the way back). |
| `hops` | array | Each hop [resolved](#resolved-hops) plus `snr_db` = how that hop heard it. |
| `final_snr_db` | number | How we heard the last one. |

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | No path, empty path, not hex, hops of different sizes, or not 1 or 2 bytes. |
| 503 | `unavailable` | The companion is not connected or refused it, or the proxy is stopping. |
| 504 | `no_answer` | A hop out of range, or a wrong path. |

### POST /api/v1/nodes/{key}/restore

Puts an archived contact back on the companion (its last contact frame: name, type, flags, path; the
companion stamps it as just modified). Works for a contact still on the companion too (it is
updated).

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/7c4d2e8f/restore
```

```json
{
  "ok": true,
  "replaced": { "key": "b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a1", "name": "Old Rover" }
}
```

| Field | Type | Meaning |
|---|---|---|
| `replaced` | object or null | The contact the companion dropped to make room (list full and overwrite oldest on); `name` may be `null`. |

| Status | Code | When |
|---|---|---|
| 404 | `not_archived` | Not in the contact archive. |
| 409 | `contacts_full` | The list is full and overwrite oldest is off (or only favourites are left). |
| 503 | `unavailable` | The companion is not connected, refused, or did not answer. |

### GET /api/v1/nodes/{key}/paths

How a node's paths changed: the path our companion uses to reach it, and the paths its adverts took
to reach us.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since` | integer | `hours=720` | Window; `hours=0` = everything. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/nodes/7c4d2e8f/paths"
```

```json
{
  "node": { "id": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
            "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
            "name": "Valley Relay", "kind": "repeater", "lat": 12.4, "lon": 45.7 },
  "current": ["a1"],
  "outgoing": [
    { "at": 1793000000, "mode": "flood", "path": [], "hops": [] },
    { "at": 1793100000, "mode": "path", "path": ["a1"], "hops": [ { "hash": "a1", "name": "Hilltop", "...": "..." } ] }
  ],
  "incoming": [
    { "from": 1793000000, "to": 1793600000, "adverts": 12, "path": ["a1"],
      "hops": [ { "hash": "a1", "name": "Hilltop", "...": "..." } ], "snr_avg_db": 4.5 }
  ],
  "incoming_paths": [
    { "path": ["a1"], "hops": [ { "hash": "a1", "name": "Hilltop", "...": "..." } ], "adverts": 40, "last": 1793600000 }
  ]
}
```

| Field | Type | Meaning |
|---|---|---|
| `current` | array or null | The `out_path` the companion holds now; `null` = none (DMs flood) or not a contact. |
| `outgoing` | array | Each change of the path our companion uses to reach it (recorded since this feature exists; the first row is the path it had then). `mode`: `"flood"`, `"direct"` or `"path"`. `hops` in path order from us outwards. |
| `incoming` | array | Periods during which its adverts reached us by the same path (`hops` from it towards us). |
| `incoming_paths` | array | The 10 most used paths. |

`hops` are [resolved hops](#resolved-hops).

## Regions

A flood message can be limited to a region (firmware v1.15 or newer). A region name is 1 to 29 of
`a-z`, `0-9`, `-` (a leading `#` is accepted; upper case is lowered). See [Regions](#regions-flood-scopes).

### GET /api/v1/regions

Every region name the proxy knows, and where from.

```bash
curl -k https://192.168.1.50:5080/api/v1/regions
```

```json
{
  "items": [
    { "name": "eu", "source": "companion" },
    { "name": "eu-se", "source": "channel" },
    { "name": "hills", "source": "repeater",
      "served_by": [ { "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "name": "Hilltop" } ],
      "fetched_at": 1793500000 }
  ],
  "next": null,
  "fetching": false
}
```

| Field | Type | Meaning |
|---|---|---|
| `source` | string | `"companion"` (its default region), `"channel"` (a channel's own region), `"user"` (added by hand), `"repeater"` (a repeater serves it). A name known from several sources keeps its first one. |
| `served_by`, `fetched_at` | array / integer | Only for names repeaters gave: which repeaters, and when last fetched. |
| `fetching` | bool | A fetch is running. |

| Status | Code | When |
|---|---|---|
| 503 | `unavailable` | The proxy is stopping. |

### PUT /api/v1/regions

Replaces the region names added by hand; answers the full list as `GET` does.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `items` | array | yes | At most 200 entries, each `{"name": "..."}` (a plain string is accepted too). Duplicates are dropped. |

```bash
curl -k -X PUT https://192.168.1.50:5080/api/v1/regions \
  -H "Content-Type: application/json" -d '{"items": [{"name": "eu"}, {"name": "eu-se"}]}'
```

The answer has the same shape as [`GET /api/v1/regions`](#get-apiv1regions).

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | No `items`, more than 200, or a bad name. |
| 503 | `no_store` | The message store is off. |
| 503 | `unavailable` | Saving failed, or no answer in time. |

### POST /api/v1/regions/fetch

Asks the repeaters in direct reach for their regions. Answered at once; `regions` events follow, the
last with `"state": "done"` (then re-read `GET /api/v1/regions`). Only repeaters stored with a
zero-hop route are asked (routes are never changed); the others are listed as `not_asked`.

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/regions/fetch
```

```json
{ "ok": true, "state": "discovering", "wait_secs": 10 }
```

| Status | Code | When |
|---|---|---|
| 409 | `busy` | A fetch is running. |
| 409 | `unsupported` | The firmware is too old to discover repeaters. |
| 503 | `radio_down` | The companion is not connected. |
| 503 | `no_store` | The message store is off. |

## Settings

### GET /api/v1/settings

The proxy settings a client may show and change.

```bash
curl -k https://192.168.1.50:5080/api/v1/settings
```

```json
{
  "hide_own_text": true,
  "mirror_sent_channel": true,
  "mirror_sent_dm": false,
  "mirror_available": true
}
```

| Field | Type | Default | Meaning |
|---|---|---|---|
| `hide_own_text` | bool | `true` | Keep the text of the messages our node sends out of the proxy's log (log files, journal, `GET /api/v1/logs`, `log` events: only its size, no hex). The live view (`activity` events, terminal app, `mcproxy watch`), `GET /api/v1/messages` and `message` events show it, and the store keeps it. Shown as one switch, e.g. "Hide my messages in the proxy's log". On the proxy machine: `mcproxy text own show / hide`; older log files: `mcproxy logs --purge-own`. |
| `mirror_sent_channel`, `mirror_sent_dm` | bool | `true` / `false` | Copies of what one device sends, for the other devices on port 5000. See [Storage and copies](#storage-and-copies). On the proxy machine: `mcproxy mirror channel / dm on / off`. |
| `mirror_available` | bool | | Read-only. `false` when the message store is off: copies cannot be made; show the two switches disabled. |

### PATCH /api/v1/settings

Changes settings: send the fields to change. Applied at once (also in the terminal app and the log)
and saved in the proxy's config; answers the new settings object. Nothing changes when one field is
wrong.

```bash
curl -k -X PATCH https://192.168.1.50:5080/api/v1/settings \
  -H "Content-Type: application/json" -d '{"mirror_sent_dm": true}'
```

```json
{
  "hide_own_text": true,
  "mirror_sent_channel": true,
  "mirror_sent_dm": true,
  "mirror_available": true
}
```

Only `hide_own_text`, `mirror_sent_channel` and `mirror_sent_dm` can be changed. Anyone allowed to
reach the API can change them: the same people can already read every message through the app port
(5000), so this adds no exposure. The allow list and the token restrict both.

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | An unknown field, or a value that is not a bool. |
| 500 | `internal` | The config could not be saved. |

## Logs

### GET /api/v1/logs

The service log, oldest first.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `lines` | integer | 500 | 1 to 10000. |
| `filter` | string | `all` | `all`, `apps`, `radio` or `problems`. |
| `grep` | string | none | Only lines containing this text. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/logs?lines=2&filter=problems"
```

```json
{
  "items": [
    { "ts": 1793600000, "level": "WARN", "text": "radio: link lost, reconnecting" },
    { "ts": 1793600012, "level": "INFO", "text": "radio: connected" }
  ],
  "next": null
}
```

| Field | Type | Meaning |
|---|---|---|
| `ts` | integer or null | Time of the line; `null` (with `level` `null`) for a line that is not a log entry. |
| `level` | string or null | `"INFO"`, `"WARN"`, `"ERROR"`, ... |
| `text` | string | The message. |

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | Unknown `filter`. |

## Companion

These are the companion's own settings: see [The companion decides](#the-companion-decides).

### PATCH /api/v1/companion/settings

Changes which contacts the companion adds by itself, and its path id size. Written to the companion
(saved in it, like the app does), then read back into `GET /api/v1/status`. Any subset of:

| Field | Type | Meaning |
|---|---|---|
| `add` | string | `"all"` or `"selected"`. |
| `chat`, `repeater`, `room`, `sensor` | bool | With `selected`: which types are added. |
| `overwrite_oldest` | bool | When the list is full, replace the oldest non-favourite contact. Also `mcproxy contacts --overwrite-oldest on / off`. |
| `max_hops` | integer | 0 to 64: 0 = any distance, 1 = heard directly only, N = up to N-1 hops. |
| `path_id_bytes` | integer | 1 to 3: the path id size our companion sends with (its path hash mode). |

See [Auto-add settings](#auto-add-settings).

```bash
curl -k -X PATCH https://192.168.1.50:5080/api/v1/companion/settings \
  -H "Content-Type: application/json" -d '{"add": "selected", "repeater": true, "max_hops": 3}'
```

```json
{ "ok": true }
```

A `status` event follows once the settings are read back.

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | Another field, a bad value. |
| 409 | `unsupported` | Firmware without these settings. |
| 503 | `unavailable` | The companion is not connected, refused, or did not answer. |

### POST /api/v1/companion/advert

Our companion sends its own advert now, like the MeshCore app's Advert buttons. Waits its turn like
any command (see [Polite sending](#polite-sending)).

| Field | Type | Required | Meaning |
|---|---|---|---|
| `flood` | bool | yes | `true` = through the repeaters; `false` = zero-hop (only nodes in range hear it). |

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/companion/advert \
  -H "Content-Type: application/json" -d '{"flood": true}'
```

```json
{ "ok": true }
```

Answered once the companion sent it.

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | No `flood`, or not a bool. |
| 503 | `unavailable` | The companion is not connected, refused, or did not answer in time. |

### PATCH /api/v1/companion/region

Sets the companion's default [region](#regions-flood-scopes), saved on the companion.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `region` | string or null | yes | A region name, or `null` = none. |

```bash
curl -k -X PATCH https://192.168.1.50:5080/api/v1/companion/region \
  -H "Content-Type: application/json" -d '{"region": "eu"}'
```

```json
{ "ok": true, "region": "eu" }
```

`region` is what the companion reads back.

| Status | Code | When |
|---|---|---|
| 400 | `bad_request` | Bad region name. |
| 409 | `unsupported` | Firmware without regions (v1.15 or newer needed), or the companion did not say its region. |
| 503 | `radio_down` | The companion is not connected. |
| 503 | `unavailable` | The companion refused it or did not answer. |

### GET /api/v1/companion/history

Our own companion over time, oldest first (one row every 15 minutes).

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Time range. |
| `limit`, `cursor` | | 500 | Paging. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/companion/history?hours=1"
```

```json
{
  "items": [
    { "ts": 1793597700, "battery_mv": 4100, "temperature_c": 6.5, "noise_dbm": -112,
      "recv": 10240, "sent": 812, "lat": 12.31, "lon": 45.62 }
  ],
  "next": null
}
```

| Field | Type | Meaning |
|---|---|---|
| `recv`, `sent` | integer or null | Packet counters since the companion booted (a drop = a reboot). |
| `noise_dbm` | integer or null | The companion's noise floor. |
| `lat`, `lon` | number or null | Its position as set in it. |

---

## Live events

### GET /api/v1/events

A Server-Sent Events stream (`text/event-stream`; in browsers `new EventSource(url)`).

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `types` | string | all | Comma-separated event names to receive, e.g. `packet,status`. `lagged` is sent whatever `types` says. |
| `token` | string | none | The API token, for clients that cannot set a header (may be percent-encoded). |

```bash
curl -k -N "https://192.168.1.50:5080/api/v1/events?types=node,status"
```

```
event: node
id: 7301
data: {"key":"a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90"}

event: status
id: 7302
data: {}

: keep-alive
```

**Stream rules**

- Each event has a name (`event:`), one JSON `data:` line and an increasing `id:`.
- A heartbeat comment is sent every 15 s.
- Clients reconnect on their own (browsers do). The stream does **not** replay missed events: a
  `Last-Event-ID` header is accepted (CORS allows it) but ignored. After a reconnect, re-read the
  REST endpoints for what you show.
- A client that reads too slowly gets a `lagged` event with the number of events that passed it by
  (the proxy buffers about 2048 events): re-read the REST endpoints, as after a reconnect. A client
  that does not know `lagged` can ignore it.
- Events nobody listens to are not built: a client sees only what happened while it was connected.

| Event | When | Payload |
|---|---|---|
| `packet` | A packet was heard. | Same object as a `GET /api/v1/mesh/packets` item, `path_nodes` included (a node heard for the first time is resolved within a minute). |
| `activity` | A readable line of what passes through (what `mcproxy watch` shows; own text hidden as configured). | `{"ts", "at", "kind", "text"}` |
| `status` | The live state changed. | `{}`: re-read `GET /api/v1/status`. |
| `node` | New data for this node (advert, status, telemetry, neighbours, owner, login), or it was added to, changed on (favourite, path, name) or removed from the companion's contact list (by any app, the companion itself, or a restore), or its "mine" mark or saved password changed. Followed by `status` when the contact count changed. | `{"key"}` |
| `contacts` | More than 50 contacts changed in one go (e.g. another companion connected, a restore). Followed by `status`. | `{"changed": n}`: re-read `GET /api/v1/nodes` instead of per-node reads. |
| `message` | A message arrived or was sent. | A `GET /api/v1/messages` item, see below. |
| `message_status` | The delivery of one of our messages changed (state, attempt, repeats). | `{"id", "client_id", "delivery"}` |
| `messages_deleted` | A DM conversation or one message was deleted from the store. | `{"with", "id", "deleted"}`: drop them from what you show. |
| `channel_messages_deleted` | A channel was removed with its messages. | `{"id", "slot", "deleted"}`: drop that channel's messages (by `id`). |
| `log` | A line was written to the service log (all levels, like `filter=all`). | Same object as a `GET /api/v1/logs` item. |
| `reset` | `mcproxy reset` wiped parts of the stored data. | `{"parts": [...]}` (`"messages"`, `"mesh"`, `"contacts"`, `"history"`): drop what you show of them and read again. Also sent with all parts when another companion connects. |
| `cli` | A node CLI command sent from here is over, by whoever sent it. | `{"id", "node", "name", "command", "reply", "state"}` (as in `GET /api/v1/nodes/{key}/cli`, secrets masked). |
| `channels` | A channel was set on the companion (added, changed or removed, by any app or the API), or a channel's own region changed. | `{"slot"}`: re-read `GET /api/v1/channels`. Or `{"channel_id", "region"}`: set through the API or the console, or learned from a MeshCore app sending in that channel with its own region (one region per channel for everyone, the last one set wins). |
| `regions` | The steps of `POST /api/v1/regions/fetch`. | `{"state": "discovering", "wait_secs"}`, `{"state": "asking"}`, then `{"state": "done", ...}`; after `done`, re-read `GET /api/v1/regions`. |
| `lagged` | This client read the stream too slowly. | `{"missed": n}` |

**The `message` event.** The same object as a `GET /api/v1/messages` item, with these differences:
`ts` is when the proxy got it; `delivery` is still `null` (`message_status` events follow for ours);
`path` and `rssi_dbm` are `null` (the packet is matched just after: read the message again, or
`GET /api/v1/messages/{id}/heard`); `text_state`, `data`, `heard_count`, `echo_of`, `path_id_bytes` and
`path_nodes` are not included. For a message sent through `POST /api/v1/messages` it carries
`client_id`, and `id` is `null` when no copy is stored.

**Example payloads**

`packet`
```json
{
  "id": 98003, "ts": 1793600420, "snr_db": 5.75, "rssi_dbm": -99, "route": "flood", "type": "advert",
  "version": 0, "hops": 1, "hash_size": 1, "path": ["a1"],
  "path_nodes": [ { "hash": "a1", "name": "Hilltop", "kind": "repeater", "confidence": 0.95, "choices": 1, "...": "..." } ],
  "last_hop": "a1", "size_bytes": 110,
  "node_key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
  "dst": null, "src": null, "channel_hash": null, "ack": null, "scoped": false, "region": null
}
```

`activity` (`at` = local time of day on the proxy; `kind` is `"app"`, `"radio"`, `"store"`,
`"frame"` or `"event"`)
```json
{ "ts": 1793600420, "at": "14:20:20.512", "kind": "radio", "text": "radio -> all  ADVERT Hilltop" }
```

`status`
```json
{}
```

`node`
```json
{ "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90" }
```

`contacts`
```json
{ "changed": 120 }
```

`message`
```json
{
  "id": 1242, "ts": 1793600430, "kind": "channel", "channel": "#hills", "channel_slot": 1,
  "channel_id": "d41e8a2c6b9f0357", "channel_kind": "hashtag", "from": "Hilltop", "from_key": null,
  "to": null, "to_key": null, "with": null, "outgoing": false, "text": "Anyone on?", "hops": 1, "snr_db": 6.0, "rssi_dbm": null,
  "path": null, "client_id": null, "delivery": null, "scoped": false, "region": null, "cli": false
}
```

`message_status`
```json
{
  "id": 1241,
  "client_id": "b7e2c9d4-1f3a-4c5e-8d7b-2a4c6e8f0a13",
  "delivery": { "state": "sent", "route": "flood", "timeout_ms": 9800, "rtt_ms": null, "at": null,
                "repeats": 0, "attempt": 1, "attempts_max": 3 }
}
```

`messages_deleted` (`with` = the key or its start as given, lowercase; or `id` for one message)
```json
{ "with": "7c4d2e8f", "id": null, "deleted": 42 }
```

`channel_messages_deleted`
```json
{ "id": "d41e8a2c6b9f0357", "slot": 1, "deleted": 95 }
```

`log`
```json
{ "ts": 1793600440, "level": "INFO", "text": "api: message b7e2c9d4-1f3a-4c5e-8d7b-2a4c6e8f0a13 sent" }
```

`reset`
```json
{ "parts": ["messages", "mesh", "contacts", "history"] }
```

`cli`
```json
{
  "id": 57,
  "node": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "name": "Hilltop",
  "command": "ver",
  "reply": "v1.9.1 (Build: 01-Jan-2026)",
  "state": "ok"
}
```

`channels` (a slot changed)
```json
{ "slot": 1 }
```

`channels` (a channel's own region changed)
```json
{ "channel_id": "d41e8a2c6b9f0357", "region": "eu-se" }
```

`regions` (start, then asking)
```json
{ "state": "discovering", "wait_secs": 10 }
```
```json
{ "state": "asking" }
```

`regions` (done)
```json
{
  "state": "done",
  "error": null,
  "found": ["eu", "hills"],
  "repeaters": [
    { "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "name": "Hilltop",
      "regions": ["eu", "hills"] },
    { "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f", "name": "Valley Relay",
      "error": "no answer" }
  ],
  "not_asked": [
    { "key": "c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2", "name": "Ridge Repeater",
      "reason": "route", "error": "its route on the companion is not zero hop (the proxy does not change it)" },
    { "key": "5d6e7f80", "name": null, "reason": "not_contact", "error": "not a contact of the companion" }
  ]
}
```

In `done`: `error` is set when the fetch ended early (the companion disconnected, or refused the
discovery); `repeaters[]` has `regions` for a repeater that answered or `error` for one that did not;
`not_asked[].reason` is `"route"` (its route on the companion is not zero hop) or `"not_contact"`
(then `key` is the short id the discovery gave and `name` is `null`).

`lagged`
```json
{ "missed": 312 }
```

---

## Concepts

### Resolved hops

A path names a repeater by the first 1-3 bytes of its key, so one id can match several repeaters.
The proxy resolves each id from context (neighbour tables, who we hear directly, distance, how
recently heard) and serves the result together with its raw inputs. Every `via`, `path_nodes`,
`out_path_nodes`, trace hop and path hop in this API is such an object:

```json
{
  "hash": "7c",
  "id": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
  "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
  "name": "Valley Relay",
  "kind": "repeater",
  "lat": 12.4,
  "lon": 45.7,
  "confidence": 0.8,
  "choices": 2,
  "could_be": [
    { "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f", "name": "Valley Relay",
      "last_seen": 1793600000, "distance_km": 9.8 },
    { "key": "7c11aa22bb33cc44dd55ee66ff7788997c11aa22bb33cc44dd55ee66ff778899", "name": "Ridge Repeater",
      "last_seen": 1793400000, "distance_km": 31.5 }
  ]
}
```

| Field | Type | Meaning |
|---|---|---|
| `hash` | string | The id as it was in the packet. |
| `id` | string | The node's key, or `"h:<hex>"` when no heard repeater has this id (`"self"` for our companion in mesh views). |
| `key`, `name` | string or null | `null` for an `"h:"` id. |
| `kind` | string | Node kind (`"unknown"` for an `"h:"` id). |
| `lat`, `lon` | number or null | Position. |
| `confidence` | number | 0..1. 0 = no heard repeater has this id. A single 1-byte match is 0.9 (it may be a repeater we never heard). |
| `choices` | integer | How many heard repeaters / rooms share the id. |
| `could_be` | array | Only when `choices` > 1: every candidate, with `last_seen` and `distance_km` (or `null`). |

How to label a hop (sure / likely / guess) is the client's choice.

### Path id size

Every hop id in a packet's path has the same size, 1 to 3 bytes (the firmware's path hash mode).
**The size is per packet**: the node that sends a packet chooses it, and every repeater adds its id at
that size (MeshCore `sendFlood` / `routeRecvPacket`). Companions default to 1 byte, so in a mesh whose
repeaters use 3 bytes, packets from companions left on the default still carry 1-byte ids. Paths in
this API are shown exactly as they came over the air.

- `/status` `radio.path_id_bytes`: the size our companion sends with (set with
  `PATCH /api/v1/companion/settings`).
- A node's `path_id_bytes` (+ `path_id_bytes_at`): the size of the packets **it** sends, read from
  its latest **flood** advert (the sender picks the size and repeaters keep it). Zero-hop adverts do
  not tell it (sent direct with path length 0, which reads as 1 byte), so a node only heard zero-hop
  stays `null`. It says nothing about how others' packets look when it forwards them: it appends its
  id at their size.
- A message's `path_id_bytes`: the size its packet really had.
- A packet's `hash_size`.

### Message paths

`path`, `path_nodes` and `rssi_dbm` of a received message are **the sender's path** (ids as in the
packet, and each one resolved, with the sender as context when known) and the signal, from the first
copy of its packet the proxy heard. They are kept by the proxy whether an app was connected or not,
so a client that reconnects later still gets them. They are `null` when the packet was not matched,
and for messages that waited on the radio while the proxy itself was not connected: the companion's
offline queue keeps only the hop count (firmware), so that path is gone everywhere. On our own
messages they are `null`: see `delivery`, and `GET /api/v1/messages/{id}/heard` for their repeats.

### SNR and RSSI

`avg_rssi_dbm` / `avg_snr_db` are plain averages over the packets. They do not add up to the noise
floor: LoRa SNR saturates around +10 dB on strong packets while RSSI keeps rising, and below the
noise floor RSSI stays near the noise floor while SNR goes negative.

### Dense hourly series

Every `per_hour` list is **dense**: one row for every hour of the range, oldest first, `ts` = the
start of the hour on a UTC hour boundary (`ts % 3600 == 0`). An hour with nothing in it has zero
counts and `null` averages (in `mesh/summary`: `{"ts", "packets": 0, "messages": 0, "avg_snr_db":
null, "by_type": {}}`). With `hours=0` the range starts at the first data stored. `per_day` lists
are dense the same way, `ts` = midnight UTC. A series never has more than 20000 rows.

### Channel ids and slots

**Channels are known by their id, not their slot.** A channel *is* its secret key (Public's is
fixed, a `#name` channel's comes from its name, a private one's is random). `channel_id` = the first
8 bytes of the SHA-256 of that key, in hex (16 characters): the same on every radio that has the
channel, in any slot; one-way, so it reveals nothing of the key (its first 2 characters are the
channel hash in packets).

- Every channel message is stored with its channel id, so it keeps its channel when channels are
  reordered or another companion is connected.
- A message whose channel cannot be told (stored by an old version without its id) has
  `channel_id: null` and belongs to no channel: never guessed from its slot, which may hold another
  channel now. It is not in `GET /api/v1/channels` and no `?channel=` finds it.
- `channel` (a name) comes from the id, whatever slot the channel is in now; `channel_slot` is the
  slot it has now. Group a conversation by `channel_id`.
- `?channel=` takes a channel id, else a name (every channel of that name), else a slot number (the
  channel in it now). `POST /api/v1/messages` takes them in the same order: send the id.
- **A send goes to its channel or nowhere.** It keeps the channel's id while it waits, and its slot is
  looked up, then read from the companion, right before it goes out. A channel that is gone by then
  is `not_found`.
- The MeshCore apps on port 5000 only know slots. The proxy keeps the list each app has read, and
  sends and passes on messages by the channel that app means, also after channels were moved by
  another app. The messages of a channel an app does not have yet are kept for it until it reads its
  channels again; everything else goes on meanwhile. A channel removed and added again with the same
  key is the same channel, in whatever slot: its history and what was kept are its own again. A message whose channel cannot be told is stored
  (`channel_id: null`) and given to no app.

### Sending without duplicates

`POST /api/v1/messages` is built so that one message goes out at most once:

- The client makes one `id` per message the user sends, never per HTTP attempt.
- The same `id` again: while it is being sent, the request waits for the same outcome; once **sent**
  or **unknown**, it gets that outcome back (for 30 minutes, and for as long as its stored copy
  exists) and nothing is sent a second time. After **not_sent** the same `id` may be sent again.
- One request = one message. A channel message is one command to the companion. A DM is resent only
  while no ACK came, as the MeshCore apps do (see [Delivery states](#delivery-states-and-dm-retries));
  the HTTP answer reflects the first attempt.
- `unknown` means it may have gone out: never resend automatically; ask the user, then send it again
  with `resend_of` (the same message, a new attempt) and a new `id`.

### Delivery states and DM retries

`delivery` on our own messages (and in `message_status` events):

```json
{ "state": "delivered", "route": "direct", "timeout_ms": 4200, "rtt_ms": 2650, "at": 1793600103,
  "repeats": 0, "attempt": 1, "attempts_max": 3, "path": ["a1", "7c"] }
```

| Field | Type | Meaning |
|---|---|---|
| `state` | string | `"sending"`, `"sent"`, `"delivered"`, `"failed"` or `"unknown"` (it was handed to the companion, which did not answer: it may have gone out). It becomes `"sent"` or `"delivered"` if an app's next attempt, or a repeat heard, shows it; it can be resent like a failed one. |
| `route` | string or null | `"flood"` or `"direct"`, as the companion sent it. |
| `path` | array or null | DMs: the path ids the attempt went by (the delivered one, else the latest), first hop first; `[]` = direct without repeater; `null` = flooded, not known, or a channel message. Per contact and route over time: `GET /api/v1/mesh/delivery`. |
| `timeout_ms` | integer or null | The companion's estimate of the ACK time (DMs). |
| `rtt_ms`, `at` | integer or null | Round trip and time of the ACK (DMs). |
| `repeats` | integer | Channel messages: repeats heard. |
| `attempt`, `attempts_max` | integer | Which attempt, of how many. |

**DMs.** `sent` once the companion sent it (`route`, `timeout_ms`), `delivered` when the recipient's
ACK came back (`rtt_ms`, `at`), `failed` when no ACK came in time (the estimate + 20 %). A late ACK
still makes it `delivered`. DMs sent through `POST /api/v1/messages` are resent like the MeshCore
apps do: up to `attempts_max` = 3 attempts, `sending` while a resend is on its way, the same
timestamp and text each time. The firmware hands every attempt that reaches the recipient to their
app, so they see one message only if that app matches timestamp and text (this proxy does). Before
the last attempt the companion forgets the route, so it goes out flooded. The ACK wait of an attempt
starts when it leaves the queue ([polite sending](#polite-sending) may hold it), and an ACK cancels
an attempt still waiting there. An attempt refused because the companion's packet table is full is
followed by the next one after a pause, instead of failing. An app resending its own DM shows as the
next `attempt` of the same message.

**Channel messages.** No ACK exists: `sent`, and `repeats` = how many times we heard repeaters pass it
on in the first 30 s (best effort: matched by channel and packet size, then by content). `repeats: 0`
a while after sending = no repeater was heard passing it on: it can be sent again with `resend_of`
(`sending`, then `sent` again, with `attempt` counting on and `repeats` counted from the resend).

A sent message is followed for 15 minutes after its last change. At start-up (and when a
companion's data comes back) DMs left `sent` or `sending` become `failed`: nothing follows them any
more.

### Resending

`POST /api/v1/messages` with `resend_of` sends a stored message again. It is the next attempt of that
message, not a new one: nothing new is stored, no `message` event follows, and
`GET /api/v1/messages` keeps one row. `delivery` goes to `sending`, then the usual states through
`message_status` on the stored id, with `attempt` and `attempts_max` counting on. `message_id` in the
answer is that stored id. Each try needs a new `id`.

**A DM** goes out with the same timestamp and text and a new attempt number, so an app that matches
those shows one message if it already has it (an ACK lost on the way back is common); others show a
repeat. It goes out once, never retried by itself, at any age. `route` works as for a new message.

**A channel message** can be resent while no repeater was heard passing it on (`repeats: 0`) and for
10 minutes after it was sent; `route` must be `auto`. It goes out as the very same packet (same
channel, timestamp and text; the companion adds the same name): nodes that already heard it drop
it as a packet they carried, so nobody shows it twice, and those that missed it get it now. Nodes
remember the last 160 packets they saw: on a very busy mesh even 10 minutes may be too long for some
of them.

| Reason | Status | Meaning |
|---|---|---|
| `not_found` | 404 | No such message sent from here; or (channel) the channel is no longer on the companion. |
| `not_contact` | 404 | (DM) The recipient is no longer a contact of the companion. |
| `not_dm` | 409 | Neither a DM nor a channel message. |
| `not_failed` | 409 | (DM) Not failed: `delivered`, or still being followed. (Channel) A repeater was heard passing it on, or it is being sent. |
| `expired` | 409 | Its text is gone, or the stored copy may be cut short (a text near the 160-byte limit with the DM copy prefix in front): send it as a new message. |
| `too_old` | 409 | (Channel) Sent more than 10 minutes ago: nodes no longer remember it, so a resend could show twice; send a new message. |
| `no_store` | 503 | The message store is off. |

### Busy retries

The companion refuses a message while its packet queue is full (16 packets; a channel message then
gets the firmware's "not found" answer, a DM "table full"). Nothing went out, so the proxy queues the
same command again after 2, 4 and 6 s before it answers `not_sent` with `reason: "busy"`. The HTTP
answer of `POST /api/v1/messages` can therefore take up to 12 s longer.

### Polite sending

Polite sending (`[polite]` in the config, on by default; `mcproxy polite on / off`) applies to every
app, the console and this API together:

- Commands that go on air (messages, adverts, requests to repeaters) go out at once up to `burst`,
  then one every `gap_secs`; meanwhile the other commands are not held up.
- At most `acks_open_max` DMs wait for their ACK at once: the companion recognises the ACKs of its
  last 8 DMs only, so one more would make it miss the oldest one's.
- DMs the proxy resends pause 1, 2, 4 s plus a random part between attempts.
- Nothing is refused for it: a request just takes longer to be answered (`POST /api/v1/messages` and
  `POST /api/v1/companion/advert` wait for it).
- An app on port 5000 sending the exact same message twice within a minute (same chat, text,
  timestamp and attempt) gets the companion's answer to the first one, and it goes out once; a manual
  resend or a DM retry is a new send and always goes out. Through this API, the same `id` already
  never sends twice.

`GET /api/v1/status` `polite` shows the settings and what waits now.

### One message, one row

A received message is known by its own id: its channel (or its sender), its sender's timestamp, its
type and its text. A copy of a message that is already stored, whenever it comes (a late repeat, a DM
its sender sent again, our own channel message coming back), is not stored again: no second row, no
`message` event, nothing to the apps on port 5000. Every client sees the same history. `heard_count`
still counts the copies heard. Messages that came twice before this was added keep both rows.

### At-least-once delivery and held ACKs

Apps on the MeshCore port (5000) get each stored message at least once: a message counts as
delivered to an app only when that app asks for the next one, so after a lost connection an app can
see a message twice. ACKs for DMs sent by an app that is no longer connected are kept for it (up to
24 hours) and handed over when it reconnects; `/status` `queue.held_acks` counts them. This API is
not limited by any of this: `GET /api/v1/messages` serves the whole stored history.

### Storage and copies

- Every message received or sent (from an app on port 5000 or through `POST /api/v1/messages`) is
  stored with its text (encrypted) and, for ours, the command it went out as.
- `mirror_sent_channel` / `mirror_sent_dm` (`[store]` in the config) only decide whether the **other apps on port 5000** get a
  copy of what one device sends. Example: a message sent on Public from a phone reaches a tablet and a
  desktop app as a received message (marked "↪ <name>"); DM copies show in the contact's conversation
  with a prefix. Off = the other apps get no copy of that kind. A generic MeshCore app therefore sees
  exactly what the companion would show it; the history, the frontends and `resend_of` see everything.
- Deleting a message in an app never deletes it from the proxy's store (other devices still get it);
  clearing a chat is a view choice of that app.
- `DELETE /api/v1/messages` and `DELETE /api/v1/messages/{id}` touch only the proxy's store: the
  companion, the contact and the copies apps already have stay. Nothing else in the store points at a
  message row, and the proxy drops what it tracked in memory for them (delivery follow-up, late
  channel labelling). Clients drop the deleted messages (and any unread state for them) from their
  own cache on `messages_deleted`; a generic MeshCore app is not affected beyond not getting them
  again on a later sync.
- Retention (texts, metadata, packets, node history, archived contacts) is set on the proxy machine
  (`mcproxy retention`); a text expired by `[store] retention_days` leaves the message with
  `text: null`, `text_state: "expired"` (the metadata stays, within `[store] metadata_retention_days`).
- Older proxies had an `[api] message_text` setting that could withhold texts when serving; the
  texts were still stored and show once the proxy is updated.

### Deduced data

Some values are deductions from what the proxy heard, not facts the radio reports:

- `knows_us`: `true` when there is proof the node has our key (it sent us a DM, or confirmed one of
  ours). A DM works only both ways round: we need it as a contact to send, and it needs us as a
  contact to read it, so "may not be able to read you" while `null`.
- `via`, `path_nodes` and every [resolved hop](#resolved-hops): a short id resolved from context, with
  `confidence`.
- `same_hash_nodes` in `GET /api/v1/nodes/{key}/heard`: relays of other nodes with the same first
  byte may be counted.
- `repeats` of our channel messages and `GET /api/v1/messages/{id}/heard`: matched by channel (or
  sender + us for DMs), size and hop count among packets heard shortly before: best effort.
- `echo_of` is exact (packet fingerprint).

### Your own nodes

`mine` = marked as yours by hand, or else your last login there gave `admin`. Set it with
`PATCH /api/v1/nodes/{key}` `{"mine": true | false | null}`: `true` adds a node, `false` keeps it off
the list even with an admin login, `null` goes back to automatic. `GET /api/v1/nodes` with
`mine=true` lists only yours. Kept per companion by the proxy, so every client sees the same list; `mcproxy reset`
leaves it alone.

### Roles and logins

`access` on a node: `{"role", "firmware", "at"}` from its last login answer, by anyone (an app too);
`null` = no login seen. `role` is `"admin"`, `"read_write"`, `"read_only"`, `"guest"`, or `null` (an
older node that does not say). `firmware` is the node's firmware level.

| Role | Commands | Status, neighbours, owner | Telemetry | Room: post |
|---|---|---|---|---|
| `admin` | yes | yes | full | yes |
| `read_write` | no | yes | full | yes |
| `read_only` | no | yes | full | no |
| `guest` | no | yes | base (battery, MCU temperature) | no |

**Saved passwords.** `"save": true` on a login keeps the password once the login worked,
`"save": false` forgets the saved one once it worked, and without `save` nothing changes. A login
that fails changes nothing. `{"use_saved": true}` logs in with the saved password.
`DELETE /api/v1/nodes/{key}/login` forgets it. Passwords are stored encrypted, per companion, and
shared by every client; no endpoint returns one (each node has `saved_login`). The proxy also uses a
node's saved password when it logs in by itself (asking for status, neighbours and so on), before its
general tracking password. `mcproxy reset --mesh` wipes them.

**CLI messages.** A command or reply sent by an app through port 5000 stays in
`GET /api/v1/messages` with `"cli": true` (apps need it): keep those out of chats, unread counts and
notifications. Replies to commands sent through `POST /api/v1/nodes/{key}/cli` are not messages at
all.

### Contact archive and other companions

**Contact archive.** The proxy keeps every contact the companion ever had (the companion holds
around 350). `left_companion_at` = when a contact left the companion's list. With every contact
listed (`contacts=all` or `hours=0`), archived contacts never heard are listed too (`contact: false`,
`favorite: null`, `out_path: null`, `heard: false`, `left_companion_at` set). How long a contact stays
after leaving: `[store] contact_archive_days` (0 = forever), `mcproxy retention --contacts N`. `/status`
shows `store.contacts_archived`, `store.contacts_archived_removed` and `store.retention.contacts_days`.
`POST /api/v1/nodes/{key}/restore` puts one back.

**Another companion connected.** The proxy knows its companion by its public key, and each companion
has its own data (messages, contacts, channels, mesh, history). When a different one connects, the
data of the previous one is set aside (`data/companions/<key>.db`) and the new one starts empty, or
gets its own data back if it was connected before. Nothing is changed on either radio. A `reset` event (all parts) and a `status`
event follow: re-read everything. A node that is a contact of the other companion shows its key in
`other_companion`. `mcproxy reset` wipes the connected companion's data.

### Auto-add settings

Which contacts the companion adds by itself (the MeshCore app's "auto add" settings), in
`GET /api/v1/status` `radio.auto_add`:

```json
{ "mode": "selected", "chat": true, "repeater": true, "room": false, "sensor": false,
  "overwrite_oldest": true, "max_hops": 0 }
```

| Field | Type | Meaning |
|---|---|---|
| `mode` | string | `"all"` = every node heard is added and the type switches do not apply; `"selected"` = only the switched-on types. |
| `chat`, `repeater`, `room`, `sensor` | bool or null | The type switches; `null` on firmware that does not report them. |
| `overwrite_oldest` | bool or null | When the list is full, the oldest non-favourite contact is replaced. Favourites never are: the firmware skips them when choosing the oldest (`BaseChatMesh::allocateContactSlot`); if every contact is a favourite, nothing is added. |
| `max_hops` | integer | 0 = any distance, 1 = heard directly only, N = up to N-1 hops. |

`auto_add` is `null` until read. The proxy keeps every node heard anyway (`GET /api/v1/nodes`, the
contact archive), whatever the companion adds. Change them with
`PATCH /api/v1/companion/settings`.

### The companion decides

The companion's settings (contacts, favourites, paths, channels, auto-add, path id size, default
region, radio parameters, name) are its own: the proxy never changes them by itself, it only passes
on changes asked through the API or the CLI. When any app changes them through the proxy (the
MeshCore app's switches, a desktop app, `mcproxy`), the proxy reads them back from the companion and
sends a `status` event, so every client shows what the radio has. A change made with the radio
connected directly to a phone is read at the proxy's next connection.

### Regions (flood scopes)

A flood message can be limited to a region (firmware v1.15 or newer; "no region" for one send needs
v1.16). The region names the proxy knows come from the companion's default region, the channels' own
regions, names added by hand (`PUT /api/v1/regions`) and names repeaters serve
(`POST /api/v1/regions/fetch`).

- The **companion's default region** is saved on the companion (`PATCH /api/v1/companion/region`).
- A **channel's own region** is kept by the proxy, the same for every app (`PATCH /api/v1/channels/{id}`,
  the console's Chat view, or learned from a MeshCore app sending in that channel with its own region;
  one region per channel, the last one set wins). `"*"` = no region, `null` = the companion's default.
  A message sent in that channel goes out in that region, then the companion's default comes back.
- Received messages and packets carry `scoped` (sent in a region) and `region` (its name, when it
  matched a known one).

### Mesh graph objects

The graph (`GET /api/v1/mesh/graph`) is built over time. Path ids are resolved to nodes as in a
[resolved hop](#resolved-hops).

**Node**

```json
{ "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "name": "Hilltop",
  "kind": "repeater", "lat": 12.3, "lon": 45.6, "hash": "a1", "confidence": 0.92, "choices": 1,
  "packets": 1200, "last_seen": 1793600400 }
```

| Field | Type | Meaning |
|---|---|---|
| `id` | string | Key, `"h:<hex>"` (an id no heard repeater has) or `"self"`. |
| `kind` | string | Node kind, or `"self"`. |
| `hash` | string or null | The path id it was seen under. |
| `confidence` | number | The average over every time a path id was resolved to it (1 for `self` and nodes only seen as origins, 0 for `"h:"` ids). |
| `choices` | integer | Heard repeaters sharing its id. |
| `packets`, `last_seen` | integer / integer or null | Packets it took part in, and when last seen. |
| `could_be` | array | Only when `choices` > 1, as in a resolved hop. |

**Link**

```json
{ "from": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "to": "self",
  "heard_forward": 800, "heard_back": 0, "snr_forward_db": 7.0, "snr_back_db": null,
  "last_seen": 1793600400, "sources": ["paths", "neighbours"] }
```

| Field | Type | Meaning |
|---|---|---|
| `heard_forward` | integer | Times `to` heard `from` (packets passed that way). |
| `heard_back` | integer | Times `from` heard `to`. |
| `snr_forward_db`, `snr_back_db` | number or null | SNR measured at the receiving side (our reception, a neighbour table or a trace); `null` when never measured. |
| `sources` | array | `"paths"`, `"neighbours"`, `"traces"`. |

One-way links and link strength are the client's calculations. Never flag a link with `"self"` as
one-way: our companion never hears its own transmissions. A path is chained only for flood packets,
never for direct ones or traces.

---

## Where each setting lives

- **Companion** = saved in the radio itself: every app sees it (through the proxy or directly over
  BLE / USB), and it survives without the proxy and a proxy reinstall. The proxy only sends the
  command.
- **Proxy** = the proxy's config file or database: only clients of this proxy see it; the radio
  knows nothing about it.
- **Client only** = each app keeps it; never sent.

| Setting / data | Stored on | Changed through |
|---|---|---|
| Contact list (who is a contact) | Companion | Apps; `POST /api/v1/nodes/{key}/restore`; `mcproxy prune` |
| Favourite of a contact | Companion (contact flags) | `PATCH /api/v1/nodes/{key}` `{"favorite"}` |
| Your own nodes (`mine`) | Proxy (per companion) | `PATCH /api/v1/nodes/{key}` `{"mine"}` |
| Saved login passwords | Proxy (per companion, encrypted, never returned) | `POST` / `DELETE /api/v1/nodes/{key}/login` |
| Path of a contact (`out_path`, its id size) | Companion (contact) | `PATCH /api/v1/nodes/{key}` `{"path"}` |
| Auto-add: all / selected, types, max hops (`radio.auto_add`) | Companion | `PATCH /api/v1/companion/settings` |
| Our path id size (`path_id_bytes`, path hash mode) | Companion | `PATCH /api/v1/companion/settings` |
| Overwrite oldest when full (`radio.auto_add.overwrite_oldest`) | Companion | `PATCH /api/v1/companion/settings` `{"overwrite_oldest"}`; `mcproxy contacts --overwrite-oldest` |
| The companion's default region | Companion | `PATCH /api/v1/companion/region` |
| Channels (names, keys, slots) | Companion (the proxy keeps a backup) | Apps; `POST /api/v1/channels`; `DELETE /api/v1/channels/{id}`; `mcproxy restore` |
| Node name, position, radio parameters (frequency, bandwidth, SF, CR, TX power), telemetry modes, advert location policy, multi-acks, BLE PIN | Companion | Apps only (passed through; no API endpoint) |
| A channel's own region | Proxy (database), the same for every app | `PATCH /api/v1/channels/{id}` `{"region"}`; the console's Chat view; learned from a MeshCore app sending in it |
| Region names added by hand | Proxy (database) | `PUT /api/v1/regions` |
| Copies of sent messages (`mirror_sent_channel`, `mirror_sent_dm`) | Proxy (config) | `PATCH /api/v1/settings`; `mcproxy mirror` |
| Own text in logs (`hide_own_text`) | Proxy (config) | `PATCH /api/v1/settings`; `mcproxy text own` |
| Polite sending (`[polite]`: on / off, burst, gap) | Proxy (config) | `mcproxy polite on` / `off` |
| What apps on port 5000 catch up on (`sync_days`, `sync_per_channel`). This API is not limited: `GET /api/v1/messages` serves the whole history | Proxy (config) | `mcproxy sync` |
| Retention (texts, metadata, packets, node history, archived contacts) | Proxy (config) | `mcproxy retention` |
| Automatic contact pruning (`prune_days`) | Proxy (config); the removals happen on the companion | `mcproxy prune --auto` |
| Tracked nodes (`[observer] track`) | Proxy (config) | `PATCH /api/v1/nodes/{key}` `{"track"}`; `mcproxy track` |
| Messages, packets, node history, neighbour / owner answers, traces, CLI history, contact archive, path timeline | Proxy (database) | Recorded |
| API token, certificate, allowed addresses, ports | Proxy (config, `~/.config/mcproxy`) | On the proxy machine only (`mcproxy api`) |
| Cleared chats, hidden / deleted messages in a view, read markers, filters, sort, layout, theme | Client only | The app |
