# Tesla Fleet API

**Status:** Active (OAuth 2.0, official partner API)  
**Base URL:** `https://fleet-api.prd.na.vn.cloud.tesla.com`  
**Auth URL:** `https://auth.tesla.com/oauth2/v3`  
**Region:** North America (`ou_code: NA`)

---

## Authentication

### OAuth Flow
- **Type:** Authorization Code
- **Client ID:** `b4a07679-8597-452d-a7c0-8a6a6b632c42`
- **Redirect URI:** `https://penndalton.com/auth/callback`
- **Scopes:** `openid vehicle_device_data vehicle_location vehicle_cmds energy_device_data offline_access`

### Scope notes (verified June 2026)
- `vehicle_device_data` no longer includes GPS — `vehicle_location` is a separate scope.
- `vehicle_cmds` is required for signed command endpoints (lock/unlock, charge_start/stop, etc.).

### API usage quota is monthly, not daily (confirmed 2026-07-24)
Tesla emails a usage-threshold alert (seen at 80%) against a **monthly** call
quota for the developer app, not a daily/hourly one. A polling design that
looks fine on a per-day basis (e.g. 30s cadence only while a wall connector
is actively charging) can still burn through the monthly allowance fast if
sustained for hours every night — a multi-hour overnight TOU charge session
polled at 30s the whole time was the actual cause here. Practical
implication: back off polling frequency once a state has been stable for a
while (see `ev-dashboard/lib/tesla.ts`'s `LIVE_STATUS_SUSTAINED_ACTIVE_MS` —
drops to a 2-min cadence after 15 continuous minutes of active charging),
don't just optimize for "fast at the moment something changes."
- **Tesla silently strips scopes not enabled in your developer.tesla.com app.** The
  consent screen won't show what's missing — you must enable them in the portal first.
- After enabling scopes you have to revoke the existing grant (or use a fresh
  account) — Tesla caches the previous scope set and reuses it on re-auth even
  with `prompt=login consent` in the URL.
- **`vehicle_location` refusal (open issue, July 2026):** Tesla will not grant
  `vehicle_location` on our app (client ID above) no matter what — portal
  checkbox enabled, re-auth, incognito, scope toggle off/on, full app revoke
  all tried; the JWT `scp` claim always comes back with only the legacy four
  scopes. Consequence: requesting the `location_data` endpoint 403s the whole
  `vehicle_data` call, so drop it from `endpoints=` (ev-dashboard does).
  Path forward is a Tesla developer-support ticket, then Fleet Telemetry
  `Location` once granted.

### Token JWT decode
The `access_token` is a JWT. Decode the payload to see what scopes Tesla
actually granted (vs. what you requested):
```sh
jq -r .access_token tokens.json | awk -F. '{print $2}' | tr '_-' '/+' |
  awk '{l=length($0); m=l%4; if(m) print $0 substr("====",1,4-m); else print}' |
  base64 -d | jq .scp
```
- **Auth endpoint:** `https://auth.tesla.com/oauth2/v3/authorize`
- **Token endpoint:** `https://auth.tesla.com/oauth2/v3/token`

### Token Exchange (Authorization Code)
```
POST https://auth.tesla.com/oauth2/v3/token
Content-Type: application/json

{
  "grant_type": "authorization_code",
  "client_id": "<client_id>",
  "client_secret": "<client_secret>",
  "code": "<auth_code>",
  "redirect_uri": "<redirect_uri>"
}
```
Response: `{ access_token, refresh_token, expires_in, token_type, id_token }`  
**Access token expires in 8 hours.**

### Token Refresh
```
POST https://auth.tesla.com/oauth2/v3/token
Content-Type: application/json

{
  "grant_type": "refresh_token",
  "client_id": "<client_id>",
  "refresh_token": "<refresh_token>"
}
```

### Request Auth Header
```
Authorization: Bearer <access_token>
```

---

## Site / Energy

### Energy Site Details
- **Site Name:** Halton Place
- **energy_site_id:** `2252299088632281`
- **asset_site_id:** `01bc8f3c-d37e-4dc4-ad95-d100c705c6a3`

### Get Site Live Status
```
GET /api/1/energy_sites/{energy_site_id}/live_status
```
Response fields:
- `solar_power` — watts
- `grid_power` — watts (negative = exporting)
- `battery_power` — watts
- `load_power` — watts
- `wall_charger_power` — watts (aggregate across all wall connectors)

---

## Wall Connectors

| Side  | device_id | Serial |
|-------|-----------|--------|
| LEFT (Rivian) | `9ded5c3b-f4ca-4061-b400-9e1591268156` | `B7S23088J08030` |
| RIGHT (Tesla) | `e4a053b8-66cd-457e-b2bc-bc41005fb45f` | `E4A23172000137` |

Part: `1457768-02-H`

### Get Energy Site Components (includes wall connector serial numbers)
```
GET /api/1/energy_sites/{energy_site_id}/components
```
**⚠️ 404s on many energy sites as of mid-2026** — Tesla appears to have
removed it for a lot of sites (including ours). Treat a 404 as permanent
for that site and stop calling (ev-dashboard sticky-disables it per
process). Wall-connector serials must then come from config/stickers;
per-connector live data comes from `live_status` (below).

Response (when it works) includes `wall_connectors` array. Each entry has:
- `device_id` — UUID used for all API calls
- `serial_number` — S/N printed on the sticker (field name TBC — check logs)
- `din` — alternate identifier field

**Known wall connector data for Halton Place (from sticker + API):**

| Side | device_id | Serial (sticker) |
|------|-----------|-----------------|
| LEFT (Rivian) | `9ded5c3b-f4ca-4061-b400-9e1591268156` | `B7S23088J08030` |
| RIGHT (Tesla) | `e4a053b8-66cd-457e-b2bc-bc41005fb45f` | `E4A23172000137` |

### Get Wall Connector Vitals (per-connector live data) — DEPRECATED

The standalone endpoint below was deprecated mid-2026 (returns 404):
```
GET /api/1/wall_connectors/{device_id}/vitals     ← 404, do not use
```

Per-connector data now lives inside `live_status` under `wall_connectors[]`:
```json
{
  "wall_connectors": [
    {
      "din": "1457768-02-H--B7S23088J08030",
      "wall_connector_state": 2,
      "wall_connector_fault_state": 2,
      "wall_connector_power": 0,
      "ocpp_status": 1,
      "powershare_session_state": 1
    },
    {
      "din": "1457768-02-H--E4A23172000137",
      "wall_connector_state": 1,
      "wall_connector_power": 11641.846
    }
  ]
}
```

- `din` — `{part_number}--{serial}`. Match against your known serial.
- `wall_connector_state` — `1` = in use, `2` = idle (observed).
- `wall_connector_power` — watts. No per-leg current or input voltage exposed
  any more; derive amps as `power / 240` for US split-phase.
- `session_energy_wh` is not available in this endpoint.

### Legacy per-connector vitals reference (no longer accessible)
```
GET /api/1/wall_connectors/{device_id}/vitals
```
Response fields (if accessible — may require additional fleet permissions):
- `vehicle_connected` — bool
- `contactor_closed` — bool (true = actively charging)
- `current_a_a` / `current_b_a` — amps per phase
- `input_voltage_v` — volts
- `session_energy_wh` — watt-hours this session

**Note:** If this endpoint returns 403, fall back to `live_status` for aggregate wall charger power.

---

## Vehicle

- **VIN:** `5YJ3E1EA3PF609276` (Tesla Model 3)

### Get Vehicle Data
```
GET /api/1/vehicles/{vin}/vehicle_data?endpoints=charge_state%3Bvehicle_state%3Bclimate_state%3Bdrive_state%3Blocation_data
```
Response: `{ response: { charge_state, vehicle_state, climate_state, drive_state, ... } }`

**Note:** `drive_state` is empty unless `location_data` is also requested (Fleet API gates GPS behind a separate endpoint name).

**charge_state fields:**
- `battery_level` — percent (0–100)
- `charge_limit_soc` — target charge limit (0–100)
- `charging_state` — `"Charging"` | `"Complete"` | `"Disconnected"` | `"Stopped"` | `"NoPower"`
- `charge_rate` — mph (add/hr)
- `battery_range` — miles
- `charge_miles_added_rated` — miles added this session
- `minutes_to_full_charge`
- `charger_actual_current` — amps
- `charger_voltage`

**vehicle_state fields:**
- `locked` — bool
- `odometer` — miles

**Important: asleep car returns a stripped response.** When the vehicle is
sleeping (`state` ≠ `"online"`), `charge_state` / `vehicle_state` /
`climate_state` come back mostly empty. If you default missing fields to
zeros/falses, you'll overwrite real cached values. Cache the last-known-good
state and restore those fields when `state` != `"online"`.

**climate_state fields:**
- `is_climate_on` — bool
- `inside_temp` — Celsius

**drive_state fields (requires `location_data` endpoint):**
- `latitude` — decimal degrees
- `longitude` — decimal degrees
- `heading` — 0-359
- `speed` — mph (null when parked)
- `shift_state` — `"P"` | `"R"` | `"N"` | `"D"` | null

### Wake Vehicle
```
POST /api/1/vehicles/{vin}/wake_up
```

### Commands
All commands: `POST /api/1/vehicles/{vin}/command/{command_name}`

| Command | Body |
|---------|------|
| `charge_start` | `{}` |
| `charge_stop` | `{}` |
| `set_charge_limit` | `{ "percent": 80 }` |
| `door_lock` | `{}` |
| `door_unlock` | `{}` |
| `auto_conditioning_start` | `{}` |
| `auto_conditioning_stop` | `{}` |
| `trigger_homelink` | `{}` (no body) — fires the car's built-in HomeLink RF transmitter |

Response: `{ response: { result: true, reason: "" } }`

**`trigger_homelink` notes** (researched 2026-06-26):

- Triggers the vehicle's HomeLink module, which transmits the same 315 MHz / 433 MHz RF signal a standard garage-door remote sends. Any HomeLink-paired opener responds — including MyQ-equipped ones — so no MyQ API account is needed for control.
- Requires the door to be **paired to the car's HomeLink module** first (one-time setup at the car: hold the HomeLink button while pressing the original remote until indicator confirms).
- Requires `vehicle_cmds` scope and the signed-command flow (so virtual key pairing via the BLE-at-the-car procedure must be done).
- Errors `not_supported` on vehicles without HomeLink hardware (some Model 3 base trims).
- **One-way only** — HomeLink can transmit but can't read door state. Pair with MyQ status polling or a tilt sensor if you need to know whether the door actually opened.
- Vehicle must be within RF range of the door (~50 ft / 15 m line-of-sight). If the car is parked off-site the call succeeds at the API layer but the door doesn't receive the signal.

---

## Vehicle Command HTTP Proxy

Modern signed-command endpoints (and `fleet_telemetry_config`) reject direct
calls to the Fleet API with:

```
"This endpoint must be called through the Vehicle Command HTTP Proxy.
 Refer to https://developer.tesla.com/docs/fleet-api/fleet-telemetry"
```

Tesla publishes the proxy as a Go binary: `github.com/teslamotors/vehicle-command`
→ `cmd/tesla-http-proxy`. It signs outgoing requests with the partner private
key and forwards to Tesla's API. Run it as a sidecar.

Build flags:
- `-tls-key` — proxy's own server TLS key (self-signed is fine if local-only)
- `-cert` — proxy's own server cert
- `-key-file` — your partner private key (`keys/private-key.pem`)
- `-port` — listen port (default 4443)
- `-host` — bind address (use `127.0.0.1` for localhost-only)

Then point your client at `https://localhost:4443/api/1/...` instead of Tesla's
real Fleet API URL. The proxy passes through unmodified for unsigned endpoints
and signs the ones that require it.

**Note:** requires Go 1.23+ to build (verified June 2026; older Go versions
fail because the upstream go.mod was bumped).

---

## Fleet Telemetry

**Status:** Active (push-based vehicle data over WebSocket).

### Wire protocol

- Vehicle establishes a **WebSocket** connection to your registered hostname.
- Path: `/` (no special path required).
- Server only receives; no response messages back to vehicle.
- **CONFIRMED (July 2026): the binary WS message is NOT raw protobuf.** The
  vehicle wraps it in a Google **FlatBuffers** envelope
  (`FlatbuffersEnvelope` → `FlatbuffersStream`, per the generated Go bindings
  in `teslamotors/fleet-telemetry`'s `messages/tesla/*.go`), and the actual
  protobuf `Payload` bytes live inside that envelope's `Payload` byte-vector
  field. You must decode the FlatBuffers wrapper first, then hand the
  extracted byte slice to your protobuf decoder. The `vin` field on the
  protobuf `Payload` isn't reliably populated in this wire format — the real
  VIN is in the FlatBuffers envelope's `DeviceId` field instead.
  Verified byte-for-byte against a real captured payload; see
  `ev-dashboard/server/telemetry-server.js` (`extractStreamMessage`) for a
  working decoder using the `flatbuffers` npm package's `ByteBuffer` export
  (note: it's `require('flatbuffers').ByteBuffer`, not a nested `.flatbuffers`
  namespace — easy to get wrong).

### Payload structure

```protobuf
message Payload {
  repeated Datum data = 1;
  google.protobuf.Timestamp created_at = 2;
  string vin = 3;
  bool is_resend = 4;
}

message Datum {
  Field key = 1;     // enum, see below for important field numbers
  Value value = 2;
}
```

### Field enum numbers (VERIFIED — Tesla has 260+ fields)

When subscribing in `fleet_telemetry_config`, use the **name** (e.g. `Soc`).
When decoding payloads, the wire uses the integer tag:

| Field name | Wire tag |
|------------|----------|
| ChargeState | 2 |
| VehicleSpeed | 4 |
| Odometer | 5 |
| Soc | 8 |
| Gear | 10 |
| Location | 21 |
| RatedRange | 32 |
| ChargeLimitSoc | 38 |
| EstBatteryRange | 40 |
| IdealBatteryRange | 41 |
| BatteryLevel | 42 |
| TimeToFullCharge | 43 |
| ChargeAmps | 49 |
| Locked | 59 |
| VehicleName | 64 |
| InsideTemp | 85 |
| OutsideTemp | 86 |
| DetailedChargeState | 179 |
| ChargerVoltage | 184 |
| HvacACEnabled | 196 |
| ChargeRateMilePerHour | 256 |

Field names that **do not exist** in the Tesla enum (don't use):
- `ChargingState` (use `ChargeState`)
- `ChargerActualCurrent` (use `ChargeAmps`)
- `ChargePortState`

### Value enum types
Important `Value` oneof fields when decoding:
- `string_value` (1), `int_value` (2), `float_value` (4), `double_value` (5), `boolean_value` (6)
- `location_value` (7) — `{ latitude, longitude }`
- `charging_value` (8) — ChargeState enum (`ChargeStateCharging = 4`)
- `shift_state_value` (9) — for `Gear`
- `detailed_charge_state_value` (32) — for `DetailedChargeState`

### Registering a config

```
POST {proxy_url}/api/1/vehicles/fleet_telemetry_config
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "vins": ["5YJ..."],
  "config": {
    "hostname": "tesla-telemetry.your-domain.com",
    "port": 443,
    "ca": "",                        // see note below
    "client_cert": "<PEM contents>", // see note below
    "fields": {
      "Soc": { "interval_seconds": 60 },
      "ChargeLimitSoc": { "interval_seconds": 300 },
      "DetailedChargeState": { "interval_seconds": 30 }
    }
  }
}
```

**`ca` field:** the trust bundle Tesla uses to validate your server's TLS cert
during the WebSocket handshake. **UPDATE (July 2026): Tesla's API now rejects
an empty string here with `"ca is not a valid PEM"`** — the old "send `""` to
fall back to the public trust store" behavior no longer works, even behind
Cloudflare with a publicly-signed cert. Fetch the actual chain your host
presents and submit the intermediate+root (strip the leaf):
```sh
openssl s_client -connect "$HOST:443" -servername "$HOST" -showcerts </dev/null 2>/dev/null \
  | awk '/-----BEGIN CERTIFICATE-----/{n++} n>1'
```
Pass that as the `ca` string. Confirmed working (`{"response":{"updated_vehicles":1}}`).

**`client_cert` field:** PEM-encoded cert Tesla will *present* to your server
during mTLS handshake. Generate a self-signed cert from your own CA.

### Required prerequisites (in order)

1. **Partner account registered.** Generate ECDSA P-256 keypair, host the public
   key at `https://<your-domain>/.well-known/appspecific/com.tesla.3p.public-key.pem`,
   POST it to `/api/1/partner_accounts`.
2. **Vehicle Command HTTP Proxy running** — signed-endpoint requests go through it.
3. **BLE virtual-key pairing** — the vehicle must have your partner public key
   enrolled via Tesla mobile app. Without this, registration responds with:
   ```json
   {"updated_vehicles": 0, "skipped_vehicles": {"missing_key": ["<VIN>"]}}
   ```
   Pair via: `https://tesla.com/_ak/<your-domain>` on a phone with the Tesla
   app, within BLE range of the vehicle.
4. **Scopes:** `vehicle_device_data` + `vehicle_cmds` (and `vehicle_location` if
   subscribing to `Location`).

### Common registration errors

| Error | Meaning |
|-------|---------|
| `This endpoint must be called through the Vehicle Command HTTP Proxy` | Not going through tesla-http-proxy |
| `Unknown field <Name>` | Field name doesn't exist in current Field enum |
| `Unauthorized missing scopes <scope>` | Token lacks scope; check JWT `scp` claim |
| `skipped_vehicles.missing_key` | Vehicle hasn't been BLE-paired with your partner key |

### Owner vs. driver account (CONFIRMED, July 2026)

Virtual key BLE pairing (and granting the "Connect Tesla" OAuth authorization
that lets a partner app control the vehicle) can only be completed by the
Tesla account that actually **owns** the vehicle. A driver-permission account
on the same vehicle can read data and send commands once already paired, but
cannot itself perform the pairing/authorize flow — the owner has to log into
Tesla's own hosted OAuth page (`https://auth.tesla.com/oauth2/v3/authorize`)
and approve it themselves, and do the BLE pairing themselves via
`https://tesla.com/_ak/<your-domain>`. There's no "add a driver to my
developer account" workaround — the developer.tesla.com app registration is
unrelated to who's allowed to grant this on any individual vehicle.

### Live confirmation of a working setup

Once telemetry is actually flowing, `docker logs` on the receiver shows
`[telemetry] connection from <car's IP>` followed by silent successful
processing (no log line unless `TELEMETRY_DEBUG=1` — success is quiet, only
decode errors/VIN mismatches log by default). The vehicle only opens this
connection while awake; a sleeping vehicle produces zero connection attempts,
which is expected, not a bug.
