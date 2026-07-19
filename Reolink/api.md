# Reolink Camera/NVR HTTP API

**Status:** Active (official, unauthenticated-by-default LAN API)
**Base URL:** `https://<camera_or_nvr_ip>/cgi-bin/api.cgi` (also reachable at `https://<ip>/api.cgi`)
**Protocol:** JSON-over-HTTP(S), single `POST` endpoint, `cmd=` query param selects the command
**Source:** Official "Camera HTTP API User Guide v8" (Reolink, via [community-maintained mirror](https://github.com/mnpg/Reolink_api_documentations))

Works the same way against a single IP camera or a Reolink NVR — the NVR just adds a
`channel` parameter to address a specific camera on it, and has a couple of NVR-only
commands (`NvrDownload`) for bulk fragment pulls.

---

## Authentication

Two modes; use long-session for anything that runs more than a couple requests.

### Long session (token) — preferred
```
POST https://<ip>/api.cgi?cmd=Login

[{
  "cmd": "Login",
  "param": {
    "User": {
      "Version": "0",
      "userName": "admin",
      "password": "xxxxxx"
    }
  }
}]
```
Response:
```json
[{ "cmd": "Login", "code": 0, "value": { "Token": { "leaseTime": 3600, "name": "42da7586d7b82a6" } } }]
```
- `Version: "0"` — leave at 0. `"1"` enables a private encryption protocol that isn't
  documented publicly; don't use it.
- Token (`Token.name`) is appended as `&token=<token>` on every subsequent request.
- **Lease is 3600s (1hr).** No refresh endpoint — just re-`Login` before/after it expires.
  Call `Logout` (`cmd=Logout&token=TOKEN`) when done to free the session slot; the device
  has a limited concurrent-session pool and stale tokens hang around until they expire.

### Short session (no login step)
Append `&user=admin&password=xxxx` directly to any command URL instead of `&token=`.
Simpler for one-off calls, but sends the password in the clear on every request — fine
on LAN over HTTPS with a self-signed cert, avoid over the open internet.

### TLS note
These devices serve a self-signed cert by default. Any client hitting this API needs to
either trust the device's cert explicitly or disable TLS verification for that one
request (`rejectUnauthorized: false` in Node, `verify=False` in requests, etc.) — don't
disable it globally.

---

## Recording history (Search → Playback/Download)

This is the flow for "show a scrollable history of clips" — three calls, in order.

### 1. `Search` — list what's recorded
```
POST https://<ip>/api.cgi?cmd=Search&token=TOKEN

[{
  "cmd": "Search",
  "action": 0,
  "param": {
    "Search": {
      "channel": 0,
      "onlyStatus": 0,
      "streamType": "main",
      "StartTime": { "year": 2026, "mon": 7, "day": 21, "hour": 0,  "min": 0,  "sec": 0 },
      "EndTime":   { "year": 2026, "mon": 7, "day": 21, "hour": 23, "min": 59, "sec": 59 }
    }
  }
}]
```
- `onlyStatus: 1` → cheap query, returns only a per-day bitmap (`Status[].table`, one
  char per day of the month, `1`=has recordings) for building a calendar view without
  pulling file lists.
- `onlyStatus: 0` → full file listing for the given range (see below). **Keep the range
  to a single day** — the guide explicitly warns wide ranges can time out.
- `streamType: "main"` vs anything else (`"sub"`) picks which encoded stream's
  recordings to search — they're recorded/retained independently.

Response (`onlyStatus: 0`):
```json
[{
  "cmd": "Search", "code": 0,
  "value": { "SearchResult": {
    "channel": 0,
    "File": [{
      "name": "Mp4Record/2026-07-21/RecM01_20260721_122057_202123_6D28C08_E4B0AE.mp4",
      "size": 14987438, "type": "main", "frameRate": 0, "width": 0, "height": 0,
      "StartTime": { "year": 2026, "mon": 7, "day": 21, "hour": 12, "min": 20, "sec": 57 },
      "EndTime":   { "year": 2026, "mon": 7, "day": 21, "hour": 12, "min": 21, "sec": 23 }
    }],
    "Status": [{ "year": 2026, "mon": 7, "table": "1111000000000011110011110000000" }]
  }}
}]
```
`File[].name` is the opaque source path you pass straight into `Playback`/`Download`.
`frameRate`/`width`/`height` come back `0` in practice — don't rely on them.

### 2. `Playback` — stream a clip (use this for the dashboard player)
```
GET https://<ip>/cgi-bin/api.cgi?cmd=Playback&source=<File.name>&output=<File.name>&token=TOKEN
```
Returns `Content-Type: application/octet-stream` with **`Accept-Ranges: bytes`** — the
device supports HTTP Range requests, so this URL is directly seekable and can be pointed
at as the `src` of a plain `<video>` element (browser issues Range requests for seeking
automatically; no need to buffer the whole file first). This is the one to use for
"scroll through history and play a clip" — `Download` (below) is the same file but
implies save-as-attachment semantics (`Content-Disposition: attachment`).

### 3. `Download` — same file, explicit save
```
GET https://<ip>/cgi-bin/api.cgi?cmd=Download&source=<File.name>&output=<local_filename.mp4>&token=TOKEN
```
Identical response shape to `Playback` but sent with `Content-Disposition: attachment`.
Use this if you want to persist a copy (e.g. mirror clips to local storage) rather than
just stream for playback.

### NVR-only: `NvrDownload` — bulk fragment pull
```
POST https://<nvr_ip>/api.cgi?cmd=NvrDownload&token=TOKEN

[{
  "cmd": "NvrDownload", "action": 1,
  "param": { "NvrDownload": {
    "channel": 0, "streamType": "sub",
    "StartTime": { "year": 2026, "mon": 7, "day": 21, "hour": 0, "min": 1, "sec": 21 },
    "EndTime":   { "year": 2026, "mon": 7, "day": 21, "hour": 0, "min": 1, "sec": 41 }
  }}
}]
```
Returns a `fileList` of small `.mp4` fragments (~seconds each, not one continuous file)
covering the requested window — only useful if you actually want the raw fragments
(e.g. to stitch/export). For UI playback, `Search` + `Playback` is simpler since it
returns full clip-length files already.

---

## Snapshot (still frame, for thumbnails)
```
GET https://<ip>/cgi-bin/api.cgi?cmd=Snap&channel=0&rs=<random_string>&token=TOKEN
```
Returns a raw JPEG. `rs` is just a cache-buster (any random string) — required on every
call or some browsers/proxies will serve a stale cached frame. Cheap way to generate a
thumbnail per `Search` result without decoding video.

---

## Practical integration plan (ev-dashboard clip history)
1. `Login` once, cache the token + a computed expiry (`now + leaseTime - buffer`);
   re-login lazily when a call 401s or the cached expiry has passed. No background
   refresh needed given the 1hr lease and this being a low-frequency admin feature.
2. Timeline/calendar view: `Search` with `onlyStatus: 1` per month to know which days
   have recordings before the user drills in.
3. Day view: `Search` with `onlyStatus: 0` for that single day → list of clips with
   start/end times and sizes.
4. Playback: point a `<video>` element straight at the `Playback` URL (append the
   cached token) — Range support means native scrubbing works without any extra
   server-side proxying or transcoding.
5. Self-signed cert: since ev-dashboard's Next.js server is the one calling the
   NVR/camera (not the browser directly, to keep the token off the client), the
   Node-side fetch needs `rejectUnauthorized: false` (or an `https.Agent` with that
   set) scoped to just this call.
