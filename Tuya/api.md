# Tuya Local LAN Protocol

**Status:** Active (local-network protocol, no cloud dependency once local_key is known)
**Protocol version:** 3.3 (confirmed for the device below; Tuya also has 3.1/3.2/3.4/3.5 variants)
**Port:** 6668/tcp (device listens; also broadcasts device info over UDP 6666/6667 for discovery)
**Library used:** [`tinytuya`](https://github.com/jasonacox/tinytuya) (Python, `pip install tinytuya`)

Applies to any whitelabeled Tuya device — the underlying hardware/firmware and
protocol are identical regardless of which skinned consumer app controls it.
This doc's example device is a Westinghouse-branded portable AC, but the
protocol section is general.

---

## Getting `local_key` + `device_id`

The skinned app (in this case Westinghouse's own app) usually can't be paired
through the real Tuya Smart / Smart Life app directly. Doesn't matter — the
device still registers on Tuya's real cloud backend regardless of app skin:

1. Create a free Tuya IoT Developer account at iot.tuya.com
2. Create a Cloud Development project, link the same account the skinned app
   uses (cloud linking works even though the consumer app is a different skin
   — same backend underneath)
3. Pull the device's `local_key` + `device_id` from the linked device list

### Cloud-link gotchas (whitelabel apps)

Linking a whitelabel device to a Tuya IoT Cloud project (via **Cloud
Development → Smart Home → Link Tuya App Account**, scanning the QR from the
skinned app's Me/settings screen) is where this usually goes wrong:

- **Region mismatch (silent failure).** The Cloud project's **data center
  region** (US/EU/China) must match the region the app account was
  **registered** in. A mismatch makes linking silently fail — the device just
  never appears, no error. Confirm the app-account's signup region first.
- **"Please use the designated app to scan the code to log in."** Tuya's
  QR-scan flow checks the scanning app against an **allowlist**. Tuya's own
  generic apps (Tuya Smart, Smart Life) are on it; whitelabel/OEM apps
  generally are NOT unless the OEM requested inclusion. The usual suggested
  fix — install Tuya Smart / Smart Life and log in with the *same account
  credentials* — **did NOT work for the Westinghouse app**: the account
  isn't actually shared/portable for that whitelabel skin (no devices
  visible under the generic app account).
- **2-project cap.** Each Tuya app account can link at most **2 Cloud
  Projects**; a 3rd needs one unlinked first.
- **Billing profile.** An unverified billing profile on the Tuya IoT
  Platform *developer* account (not the app account) can also block linking,
  even during the free trial.

### Shortcut: pull `local_key` from an existing local integration

Before fighting the Cloud-Project QR-link flow, **check for an existing
Homebridge / other local integration that already has the device's
`id`/`key`/`ip`** (e.g. a `homebridge-tuya`/Tuya Plus plugin config) — it may
already be solved. Gotcha when verifying a recovered key: `tinytuya` can fail
at every protocol version (3.1–3.5) even with a byte-correct key if
**another integration is already holding the device's single local
connection slot** (most Tuya devices allow only one). Disable that other
integration first, then connect.

## Connecting locally

```python
import tinytuya
d = tinytuya.Device(device_id, local_ip, local_key)
d.set_version(3.3)  # try 3.3, 3.4, 3.2 if unsure
status = d.status()   # -> {"devId": "...", "dps": {"1": true, "6": 24, ...}}
d.set_value(1, False)  # write a single DP
```

No TLS, no cloud round-trip once you have id/ip/key — this is a local
encrypted-with-local_key protocol, decrypted transparently by `tinytuya`.

## Gotchas

- **One local session at a time.** If the vendor's app has an active
  connection (open, or briefly after making a change there), local queries
  fail with `{"Error": "Network Error: Unable to Connect", "Err": "901"}`.
  Close the app before scripting against the device.
- **Rate limiting / throttling.** Repeated rapid connection attempts (tens of
  requests over ~20 min) triggered a period where every request failed the
  same way as above. Back off 60-90s and retry once rather than hammering it.
- **`updatedps()` (force a broader DP refresh) is often unsupported** —
  returns `{"Error": "Timeout Waiting for Device", "Err": "902"}` and reveals
  nothing beyond the DPs already returned by a normal `.status()` call. Don't
  rely on it to discover hidden DPs.
- **DP numbers are not self-describing** — must be reverse-engineered per
  device/product by diffing `.status()` output against real actions taken via
  the vendor's app (or Tuya IoT Cloud's Device Debug panel, not tried here).
  UI *display order* in the vendor app does not necessarily match the raw
  enum values Tuya uses on the wire — confirm by diffing, don't assume.

## Example: Westinghouse WPAC12000S portable AC

Device model: **WPAC12000S** — whitelabeled Tuya, sold under the Westinghouse
brand; the Westinghouse app blocks pairing through the real Tuya app.

### DP (datapoint) map — confirmed 2026-08-14

| DP | Meaning | Values | Confidence |
|----|---------|--------|------------|
| `1` | Power | bool | Confirmed |
| `5` | Mode | string enum: `"3"`=Cool | Cool confirmed; Auto/Dry/Fan values NOT confirmed (UI shows those 4 options but only Cool was live-tested) |
| `6` | Target temp, °C | int | Confirmed — authoritative/live value, always in Celsius regardless of display unit |
| `8` | Fan speed | string enum: `"0"`=Auto, `"1"`=Low, `"2"`=Medium, `"3"`=High | Confirmed (0/1/3 tested live; 2 inferred from UI ordering, high confidence) |
| `10` | Display unit preference | bool: `true`=Fahrenheit, `false`=Celsius | Confirmed |
| `16` | Oscillation | bool | Confirmed |
| `17` | Sleep mode | bool | Confirmed |
| `18` | Target temp, °F | int | Confirmed, but **not a live mirror of DP 6** — only recalculated from DP 6 when DP 10 (unit preference) is toggled. Conversion isn't exact standard rounding (20°C→68°F expected, device produced 69) — treat as device-computed, not something to replicate client-side. |
| `4`, `13`, `14`, `15`, `19` | Unknown | `4`/`13`/`14`/`15` are ints (seen only as `0`), `19` is bool (seen only as `false`) | **Unmapped.** Never observed to change across power/mode/speed/temp/sleep/oscillation/unit toggles. Likely candidates: child lock, filter/clean reminder, buzzer/beep, WiFi signal indicator, or timer/countdown state — but Tuya's "Add Timer" scheduling feature in the vendor app appears to be cloud-side (server-triggered normal commands at trigger time), not backed by local DPs, so it's unlikely to explain these. Confirming these requires either (a) Tuya IoT Cloud's Device Debug panel/TSL schema, or (b) blind `set_value()` writes to each while physically watching/listening to the unit for a reaction — not yet attempted. |

### Behavior notes

- Switching the display unit (`10`) causes the AC to reset the actual target
  setpoint (`6`) to a default value, not just reformat the display — e.g.
  switching C→F→C mid-session did not restore the original temp. Don't toggle
  `10` programmatically unless you intend to also re-set `6` afterward.
- The vendor app's temp dial can show a stale/default value (`69°F`) briefly
  on fresh load before syncing to the device's real state — don't trust the
  first frame after opening the app.

### Credentials

Device-specific `device_id`/`local_key`/LAN IP for this specific unit are
**not** stored here — see the private vault note at
`Claude/Projects/westinghouse-homebridge.md` for those.

### Homebridge integration

Once local_key is known, `homebridge-tuya-local` (existing, maintained
Homebridge plugin) or a direct `tinytuya` script can drive the device using
the DP map above. Power/mode(Cool)/temp/fan-speed alone are enough to back
HomeKit's HeaterCooler service.
