# SolarEdge

Two paths to read inverter data:

1. **Local Modbus/TCP** (recommended for live data)
2. **Cloud Monitoring API** (`monitoringapi.solaredge.com`, capped at 300 polls/day)

---

## Local Modbus/TCP

Direct LAN polling of the inverter — no cloud, no rate limit, sub-second granularity possible. Cloud monitoring keeps working alongside it.

### Enable on the inverter (one-time)

SetApp inverters (no LCD — most modern HD-Wave / HD-Wave 2 / Single Phase Inverters):

1. Flip the red P/1/0 toggle to **P** for **< 5 seconds**, then back to **1**. WiFi Direct comes up.
2. Join `SE-WiFiDirect_<serial>` (password printed on the inverter's side label).
3. Open SetApp app, or browse to `http://172.16.0.1` from a laptop.
4. Login as **Installer**, default password `12312312` (call SolarEdge installer support if the installer changed it).
5. Site Communication → Modbus TCP → **Enable**.
6. Port: **1502** (default; not 502 — SolarEdge picked a non-standard port).
7. Device ID: **1** (default).
8. Power-cycle the toggle back to 1. Port stays open on the inverter's LAN IP.

LCD inverters: enter installer mode (hold OK 5 sec) → Communications → LAN → Modbus TCP.

### Critical constraints

- **First connect within 2 minutes** of saving the setting — otherwise the port closes and you redo the SetApp menu.
- **Single Modbus client at a time.** Home Assistant, Energy Manager, or any other Modbus consumer will fight ours. Disable other consumers before connecting.
- Cloud monitoring keeps working — the Modbus port and the cloud uplink are independent.

### SunSpec model 103 (single-phase inverter) register layout

Base address: SunSpec uses 1-based register numbers; Modbus wire protocol is 0-based, so subtract 1.

| SunSpec | Field | Type | Scale factor (signed int16, sets `× 10^SF`) |
|---------|-------|------|--------|
| `40069` | Model block start | — | — |
| `40083` | AC Current (A) | uint16 | `40084` |
| `40092` | AC Power (W) | sint16 | `40093` |
| `40094` | Frequency (Hz) | uint16 | `40095` |
| `40100..40101` | Lifetime AC Energy (Wh) | uint32 | `40102` |
| `40103` | DC Current (A) | uint16 | `40104` |
| `40105` | DC Voltage (V) | uint16 | `40106` |
| `40107` | DC Power (W) | sint16 | `40108` |
| `40111` | Operating State | uint16 (enum) | — |
| `40117` | Status Code | uint16 (vendor) | — |

**Operating State enum:** 1=off · 2=sleeping · 3=starting · 4=mppt · 5=throttled · 6=shutting down · 7=fault · 8=standby.

**Daily energy** is NOT a register — derive from the lifetime counter (midnight snapshot, subtract).

### Function codes used

FC03 (Read Holding Registers) is sufficient for all reads above. Request format: `txId(2) protoId(2)=0 length(2)=6 unitId(1) fc=0x03 startAddr(2) qty(2)`. Response: same header + byteCount + N×2 bytes (big-endian).

### Reference docs (current as of 2026-06-26)

The standalone `se-modbus-tcp-application-note.pdf` was retired. Current sources:

- **SunSpec Implementation Technical Note** (June 2025) — register map + Modbus TCP config:
  `https://knowledge-center.solaredge.com/sites/kc/files/sunspec-implementation-technical-note.pdf`
- **Communication Options Application Note** (firmware 2.50+) — SetApp menu walkthrough:
  `https://knowledge-center.solaredge.com/sites/kc/files/solaredge-communication_options_application_note_v2_250_and_above.pdf`

---

## Cloud Monitoring API

`https://monitoringapi.solaredge.com/`

**Auth:** 32-char `api_key` query param on every request. Per-site key.

**Where the key comes from:** `monitoring.solaredge.com → Admin → Site Access → API Access`. **Requires site admin role.** If the Admin tab is missing, you're not admin OR you're on the new "ONE" UI — click "Switch to Classic" in the top-right.

**Rate limits:** 300 requests/day per account + 300/day per site. Caps practical polling at ~5-minute granularity (288 polls/day).

### Useful endpoints

| Endpoint | Returns |
|----------|---------|
| `GET /site/{siteId}/overview?api_key=…` | Lifetime / yearly / monthly / daily energy + current power |
| `GET /site/{siteId}/currentPowerFlow?api_key=…` | Live PV → grid / loads / battery flow (W) |
| `GET /site/{siteId}/power?startTime&endTime&api_key=…` | 15-min power buckets |
| `GET /site/{siteId}/energy?timeUnit=DAY&api_key=…` | Daily kWh series |
| `GET /equipment/{siteId}/{serial}/data?api_key=…` | Per-inverter telemetry |

No `v1` / `v2` URL split — SolarEdge has only the one REST surface.

---

## Integration files

| File | Purpose |
|------|---------|
| `ev-dashboard/lib/solaredge.ts` | Zero-dependency Modbus/TCP FC03 client + SunSpec model 103 parser |
| `ev-dashboard/app/api/solaredge/live/route.ts` | GET endpoint returning the cached live snapshot |
| `ev-dashboard/SOLAREDGE-INTEGRATION.md` | Path A vs Path B research + recommendation |
| `ev-dashboard/SOLAREDGE-MODBUS-ENABLE.md` | At-the-inverter print sheet |
