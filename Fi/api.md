# Fi (TryFi / fitracking) — GPS smart dog collar

Unofficial notes. Fi has **no official/public API or developer docs**. Everything below is either
from Fi's own help center (authoritative, ✓) or from community clients / observed behavior
(~ = not live-verified here). Support domain moved `tryfi.com` → `fitracking.com` (301).

Evaluated 2026-09-11 for a third-party Base build; scrapped because the collar does WiFi natively
(see WiFi section). This doc is the retained knowledge.

## Architecture

Collar (Series 2 / 3 / Ultra) + optional Fi Base + phone app + Fi cloud.

**Connection sources** ✓ (Fi help "View Current Device Connection") — the app reports current /
last source as one of:
- **Fi Base** — collar reaching its Base over BLE; Base relays to cloud over WiFi.
- **Wi-Fi** — collar joined a saved WiFi network directly (see below).
- **Phone** — collar connected to the owner's phone over Bluetooth.
- **GPS / Cellular** — LTE-M + GPS, when the dog leaves known zones (battery-intensive).
- **Satellite** — Fi Ultra only.

"Home" is a known-zone state reachable via **either** a Base (BLE) **or** a saved WiFi network;
LTE-M+GPS is the expensive fallback when out of all known zones.

## Cloud API (unofficial GraphQL)

- **Auth** ~ — app-account **email + password** login yields a session/token for subsequent calls
  (how `pytryfi` authenticates; it retrieves an API-specific session after login).
- **Transport** ~ — GraphQL under `api.tryfi.com` per community clients; treat the exact URL/schema
  as whatever `pytryfi` currently uses (undocumented, can change).
- **Exposed per `pytryfi`** ~ — current user; pets (device battery %, charging, firmware, current
  location lat/long+place, activity/rest stats, lost / Lost-Dog-Mode, LED on/off+color); bases
  (online status). Use `pytryfi` as source of truth for exact field/method names.
- **Rate limits / session longevity** — undocumented.

## WiFi networks (collar-native — the important part)

✓ Fi help "How Do I Add/Remove/Edit Wi-Fi Networks":
- The **collar joins WiFi directly — no Base required**: *"This allows you to accurately track your
  pet indoors without needing a Fi Base."*
- **Up to 6** saved networks per device. **2.4 GHz** only. **Series 2/3+** (Series 1 cannot).
- Added with just **SSID + password**, but **must be in range with the collar present** at add-time
  (the collar joins + confirms). Some captive portals (hotels) fail.
- Adding a network prompts creating a **Safe Zone** → WiFi presence is tied to a location/home.
- App flow: pet avatar (lower-right) → ☰ → **Your Pack** → Wi-Fi section → **Add network** → pick
  from scan (or **Add manually**) → password → set location on map → create Safe Zone.
- Whether networks can be added via the cloud API is **unconfirmed** (~) — provisioning appears
  app/in-range only.

## Fi Base

- **Charger** — magnetic pogo-pin cradle + USB; collar pulses blue. Charging cradle is
  **series-specific** (S2 collar ≠ charge on S3 Base and vice-versa) ✓.
- **BLE link + WiFi backhaul** — connects to collar over Bluetooth, relays to cloud over 2.4 GHz
  WiFi; ~150 ft; sold as range extenders. **All collars pair with all Bases** for connectivity
  (only charging is series-specific) ✓.
- **Base protocol** (BLE collar↔Base, Base→cloud) is **proprietary/unpublished**.

## Prior art / clients

- **`pytryfi`** — Python GraphQL client (PyPI `pytryfi`; `github.com/sbabcock23/pytryfi`).
- **Home Assistant** — `github.com/tryfi/hass-tryfi` (Fi's own org; embeds pytryfi), `mdb17/hacs-fi`,
  `sbabcock23/hass-tryfi`, `KTibow/hass-tryfi`.
- `github.com/tryfi` publishes **only** the HA integration — not firmware/app/backend source.

## Gotchas

- Undocumented API → can change without notice; community clients periodically break on Fi app
  updates. Auth is your normal Fi login (no separate API key).
- For "home / battery-saving" without buying extra Bases: just add the relevant **WiFi SSIDs** (up
  to 6). A physical Base only adds value where the collar **can't reach any WiFi** (true dead
  spots) — that BLE-bridge role is the only thing a WiFi network doesn't replace.
