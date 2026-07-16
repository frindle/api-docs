# Firewalla API

Firewalla Gold Plus at `10.0.0.1`. Multiple API surfaces exist; only the local API via SSH is used by network-bandwidth-monitor.

## Access Methods

### 1. Local API via SSH (primary — what we use)

`app-local.js` listens on `127.0.0.1:8834` — **not LAN-accessible**. Access by SSHing into the Firewalla and curling locally:

```bash
ssh pi@10.0.0.1 "curl -s http://127.0.0.1:8834/v1/host/all"
```

- SSH user: `pi`, port 22
- SSH key: `/root/.ssh/id_firewalla` (in our container)
- No auth token needed for the local API
- CORS-only — no authentication headers required

### 2. Consumer Cloud API (my.firewalla.net) — FREE

**Base URL:** `https://my.firewalla.net`
**Auth:** QR-code mobile app approval (no username/password login)
**Discovered:** 2026-07-14 via API spy on the web console

Available to all Firewalla owners — no paid tier. This is the same API backing the web management console. Has per-device bandwidth data with actual byte volumes that the local API doesn't expose. See "Consumer Cloud API Endpoints" section below.

**Auth flow (confirmed 2026-07-14):**
1. `GET /v1/sandbox/prepare` — returns `{token, ts, expire}` (token = UUID, expire = ts + 86400 = 24h)
2. Browser displays QR code encoding the token
3. User scans QR with Firewalla mobile app to approve
4. Browser polls `GET /v1/auth/{token}` every 5s — returns `{}` until approved
5. Once approved, session cookie is set and all API endpoints work

**Automation blocker:** No username/password path. Every new session requires scanning a QR code with the Firewalla mobile app. The login token expires in exactly 24 hours (`expire - ts = 86400s`). Session cookie lifetime after approval is unknown — could be longer than 24h but untested. This makes the cloud API impractical for automated polling in netmon without daily manual QR scan.

### 3. MSP Cloud API (paid tier required)

**Base URL:** `https://<msp_domain>.firewalla.net`
**Docs:** https://docs.firewalla.net/
**Auth:** Personal access token in `Authorization` header

Not currently used — requires paid MSP subscription. Different from the consumer cloud API above.

### 4. Encipher/Internal Box API (port 8833)

LAN-accessible at `http://10.0.0.1:8833` but uses an encrypted message protocol:
- ETP tokens (public/private keypair from QR code pairing)
- All messages AES-encrypted with symmetric key from pairing handshake
- Routes through `firewalla.encipher.io` relay even for "local" access
- NOT a simple REST API — too complex for our needs

Library: https://github.com/lesleyxyz/node-firewalla
Token generator: https://github.com/lesleyxyz/firewalla-tools

### 5. Redis (direct SSH)

```bash
ssh pi@10.0.0.1 "redis-cli hgetall host:mac:AA:BB:CC:DD:EE:FF"
```

Used for device management (naming, group assignment), not data collection.

---

## Local API Endpoints

### `GET /v1/host/all`

All devices known to the Firewalla.

**Response:**
```json
{"hosts": [...]}
```

**Host fields:**

| Field | Type | Description |
|-------|------|-------------|
| `mac` | string | MAC address |
| `ip` | string | Current IP |
| `name` | string | Device name (user-assigned or auto-detected) |
| `localDomain` | string | mDNS/local hostname |
| `macVendor` | string | Manufacturer from OUI lookup |
| `lastActive` | number | Unix timestamp of last activity |
| `type` | string | Device category: "tv", "tablet", "smart speaker", etc. |
| `flowsummary.inbytes` | number | Cumulative download bytes |
| `flowsummary.outbytes` | number | Cumulative upload bytes |
| `topDownload` | array | Top connections by download — **counts only, no byte volumes** |
| `topUpload` | array | Top connections by upload — **counts only, no byte volumes** |

### `GET /v1/flow?begin=<ts>&end=<ts>&count=<n>`

Flow/connection records for a time range.

**Status:** Returned 404 in earlier testing (2026-05). Wired up in `fw_flows_collector.py` but may not work on all firmware versions.

**Response (when working):**
```json
{"flows": [...]}
```

**Flow fields (observed, names vary across firmware):**

| Field | Aliases | Description |
|-------|---------|-------------|
| `sh` | `src`, `sourceIP` | Source IP (LAN device) |
| `dh` | `dst`, `destIP`, `rh` | Destination IP (remote host) |
| `dn` | `domain`, `hostname`, `h` | Domain name (DNS/SNI) |
| `ob` | `upload`, `tx`, `sb` | Upload bytes |
| `rb` | `download`, `rx` | Download bytes |
| `pr` | `protocol` | Protocol (6=TCP, 17=UDP) |
| `dp` | `dport`, `port`, `p` | Destination port |
| `fd` | `direction` | Flow direction: `out`, `in` |
| `ts` | `timestamp` | Unix timestamp |

### `GET /v1/stats?begin=<ts>&end=<ts>`

Aggregate stats for a time range. Response shape not fully documented.

---

## Consumer Cloud API Endpoints (my.firewalla.net — free)

Box GID: `acc6cec5-51ea-4625-ab03-906c7d10bf94`

### `GET /v1/device/list`

All devices with cumulative bandwidth. Returns an array directly (no wrapper object).

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Device name |
| `ip` | string | Current IP |
| `mac` | string | MAC address |
| `gid` | string | Box ID |
| `intf` | object | `{uuid, name}` — network interface |
| `macVendor` | string | Manufacturer |
| `online` | boolean | Current connectivity |
| `totalDownload` | number | Cumulative download bytes |
| `totalUpload` | number | Cumulative upload bytes |
| `lastActive` | number | Unix timestamp |
| `firstFound` | number | Unix timestamp |
| `tags` | array | `[{name, uid}]` — group memberships |
| `deviceType` | string | `"switch"`, `"nas&server"`, `"tv"`, `""` |
| `allocation` | string | `"dynamic"` or `"static"` |
| `group` | string | Group UID |
| `stpPort` | string | Physical port (e.g. `"eth2"`) |
| `hostname` | string | DHCP/mDNS hostname (when available) |

### `GET /v1/device/<mac>?gid=<gid>`

Single device detail with **hourly trend data** — the big win over the local API.

Additional fields beyond `/v1/device/list`:

| Field | Type | Description |
|-------|------|-------------|
| `reservedIP` | string | Static IP reservation |
| `hostname` | string | DHCP hostname |
| `policy` | object | Per-device policies (monitor, DAP state) |
| `allowedPeers` | array | Allowed peer devices |
| `trends.24h` | array | Hourly bandwidth breakdown (see below) |

**Trend entry fields:**

| Field | Type | Description |
|-------|------|-------------|
| `ts` | number | Hour start (Unix timestamp) |
| `count` | number | Connection count |
| `download` | number | Download bytes this hour |
| `upload` | number | Upload bytes this hour |
| `total` | number | Total bytes this hour |
| `blocked` | number | Blocked connection count |

### `GET /v1/device/<mac>/trend?gid=<gid>&interval=24h&timezone=America/Los_Angeles`

Dedicated trend endpoint — same hourly data as the `trends` field in device detail. Returns array of trend entries directly.

Params: `interval` (`24h` observed), `timezone` (IANA timezone string).

### `GET /v1/device/<mac>/rules?gid=<gid>`

Per-device block rules (auto-generated from intel/alarms).

| Field | Type | Description |
|-------|------|-------------|
| `pid` | string | Policy/rule ID |
| `action` | string | `"block"` |
| `type` | string | `"ip"` |
| `target` | string | Blocked IP |
| `target_name` | string | Blocked IP or domain |
| `method` | string | `"auto"` (intel-triggered) |
| `category` | string | `"intel"` |
| `hitCount` | number | Times rule was triggered |
| `lastHitTs` | number | Last hit timestamp |
| `lastHitFlow` | object | Last flow that triggered the rule |
| `scopeInfo` | object | `{type, name, id, model}` |
| `blockon` | string | `"global"` or device-scoped |

### `GET /v1/device/<mac>/alarms`

Device alarms with transfer byte volumes and geo info.

| Field | Type | Description |
|-------|------|-------------|
| `aid` | number | Alarm ID |
| `_type` | string | `"ALARM_LARGE_UPLOAD"`, `"ALARM_INTEL"`, etc. |
| `ts` | number | Alarm timestamp |
| `status` | number | Alarm status |
| `device` | object | `{id, name, ip, macVendor, group, network, deviceType}` |
| `remote` | object | `{ip, region, longitude, latitude, port}` |
| `transfer` | object | `{upload, download, total, duration}` — bytes |
| `direction` | string | `"inbound"`, `"outbound"` |
| `message` | string | Human-readable alarm description |

### `GET /v1/networkAndGroup/list`

All groups with full device membership and policies. Returns array of groups, each with `devices` array containing full device objects (same fields as `/v1/device/list` plus `policy`, `detect`, `deviceTags`, `userTags`).

### `GET /v2/users?scope=box:<gid>`

User profiles with affiliated device MACs and WireGuard peers.

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | `<gid>:<user_id>` |
| `name` | string | User name |
| `status` | boolean | Active |
| `devices` | array | MAC addresses + `wg_peer:` entries |
| `affiliatedTag` | string | Group UID |
| `rules` | array | User-specific rules |
| `policy` | object | User-specific policies |

### `GET /v1/rule/targets?scope=box:<gid>`

All devices with full policy and DAP (Device Access Protection) rules. Response: `{devices: [...]}`.

### `GET /v2/boxes/<gid>/dataset?fields=<comma-separated>`

Mega-endpoint. Known fields: `devices`, `networkConfig`, `networkProfiles`, `upnpRules`, `aps`, `snatRules`, `nicStates`, `policy`, `portforward`, `inboundAllowRules`, `dhcpLeaseInfo`, `scanUpnp`, `vpnClients`.

Returns requested fields as top-level keys. `devices` array contains same data as `/v1/device/list`.

### Auth Notes

The consumer cloud API uses browser session cookies set after Firewalla app login. To automate:
1. Need to reverse-engineer the session/token flow (likely encipher-based under the hood)
2. Or extract cookies from an authenticated browser session
3. Or use the encipher pairing flow programmatically via `firewalla-tools`

Not yet integrated into network-bandwidth-monitor. The session auth is the main blocker — SSH+local API is simpler for automated polling.

---

## MSP Cloud API Endpoints (reference only — paid)

### `GET /v2/devices`

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | MAC address |
| `macVendor` | string | Manufacturer |
| `gid` | string | Box ID |
| `ip` | string | IP address |
| `name` | string | Device name (max 32 chars, editable via PATCH) |
| `online` | boolean | Current connectivity |
| `lastSeen` | number | Last activity timestamp |
| `network` | object | `{id, name}` |
| `group` | object | `{id, name}` |
| `totalDownload` | number | Cumulative download bytes |
| `totalUpload` | number | Cumulative upload bytes |

### `GET /v2/flows`

Query params: `query`, `groupBy`, `sortBy`, `limit` (max 500, default 200), `cursor`

| Field | Type | Description |
|-------|------|-------------|
| `ts` | number | Flow end timestamp |
| `protocol` | string | `tcp` or `udp` |
| `direction` | string | `outbound`, `inbound`, `local` |
| `block` | boolean | Whether traffic was blocked |
| `download` | number | Bytes downloaded (regular flows only) |
| `upload` | number | Bytes uploaded (regular flows only) |
| `duration` | number | Session duration in seconds |
| `category` | string | Traffic classification (e.g. "shopping") |
| `count` | number | Connection count |
| `region` | string | ISO 3166 country code |
| `device` | object | `{id, ip, name}` |
| `destination` | object | `{id, ip, name}` |

---

## Redis Data Model

### Device records

Key: `host:mac:<MAC>` (hash)

Fields include: `name`, `ip`, `macVendor`, `lastActiveTimestamp`, and many others.

### Group/policy tags

Key: `policy:mac:<MAC>` (hash)

The `tags` field is a JSON array of group IDs:
```
["5"]       # Home Automation
["35"]      # Quarantine
```

### Pubsub channels

- `Host:Updated` — device state changes
- `Host:PolicyChanged` — policy/group changes

Publish after Redis writes to notify Firewalla of changes:
```bash
redis-cli publish "Host:PolicyChanged" '{"mac":"AA:BB:CC:DD:EE:FF"}'
```

### Known group IDs

| ID | Name |
|----|------|
| 5 | Home Automation |
| 35 | Quarantine |

---

## SSH Data Sources

### WAN interface counters

Read `/proc/net/dev` on the Firewalla for per-interface byte counters:

```bash
ssh pi@10.0.0.1 'grep -E "eth0:|eth3:" /proc/net/dev'
```

| Interface | WAN |
|-----------|-----|
| `eth0` | Cox (primary) |
| `eth3` | Starlink |

Fields: standard `/proc/net/dev` format — column 1 = rx_bytes, column 9 = tx_bytes.

---

## Integration Files

| File | Purpose |
|------|---------|
| `app/firewalla.py` | SSH+curl client for local API |
| `app/fw_collector.py` | Device sync every 5 min via `/v1/host/all` |
| `app/fw_flows_collector.py` | Flow polling every 5 min via `/v1/flow` |
| `app/starlink_collector.py` | WAN counters every 30s via SSH `/proc/net/dev` |

## Known Limitations

- **Local API:** `topDownload`/`topUpload` per-device return connection counts only, no byte volumes (consumer cloud API has byte volumes via `/v1/device/<mac>/trend`)
- **Local API:** `/v1/flow` returned 404 on some firmware versions — may not be universally available
- `socat` not installed on Firewalla (can't use for port forwarding)
- Local API field names vary across firmware versions (hence alias handling in `fw_flows_collector.py`)
- **Consumer cloud API:** Session auth not yet automated — main blocker for integration
