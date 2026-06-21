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
- **Scopes:** `openid vehicle_device_data energy_device_data offline_access`
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

### Get Wall Connector Vitals (per-connector live data)
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
GET /api/1/vehicles/{vin}/vehicle_data?endpoints=charge_state%3Bvehicle_state%3Bclimate_state
```
Response: `{ response: { charge_state, vehicle_state, climate_state, ... } }`

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

**climate_state fields:**
- `is_climate_on` — bool
- `inside_temp` — Celsius

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

Response: `{ response: { result: true, reason: "" } }`
