# Deluge (JSON-RPC / WebUI)

Instance: `http://<lan-ip>:8112`, Unraid container `binhex-deluge`. Version confirmed 2.2.0.
Auth: `POST /json` `{"method":"auth.login","params":[PASSWORD],"id":1}` — session cookie on the requests.Session.
All calls are `POST /json` with `{method, params, id}`.

## Verified RPC semantics (checked against the shipped deluge-2.2.0 package, 2026-08-06)

Verify claims against `deluge/core/torrent.py` + `deluge/core/core.py` in the wheel
(`pip3 download deluge`, unzip) — not just the docs, which are thin here.

### core.move_storage(torrent_ids: list, dest: str)
The ONLY primitive that relocates an existing torrent's data. Calls libtorrent
`handle.move_storage(dest, flags=2)`. **flags=2 = `dont_replace`**, whose real
semantics per Deluge's own inline comment: *"dont overwrite target files but
delete source"*. So:
- Files already present at `dest` → kept, source copies **deleted**.
- Files only at the source → **moved** to dest.
- Creates `dest` (os.makedirs) if missing.

This is genuinely destructive. It is the correct "relink a torrent to data that
moved" call, but it moves/deletes real bytes — never fire it speculatively.

### core.set_torrent_options(torrent_ids, {'download_location': ...})
**NOT a relink.** `Torrent.set_download_location` only does
`self.options['download_location'] = value` — it never touches existing data.
Using it to "point a torrent at a new path" leaves the torrent broken while
looking reconfigured. (A common wrong suggestion; don't regress to it.)

### core.force_recheck(torrent_ids: list)
Re-verifies pieces against whatever is on disk, **and resumes internally**
(`handle.force_recheck(); handle.resume()`), so no separate `core.resume_torrent`
is needed. Moves no data → safe and idempotent. This is the right first response
to a post-power-outage `Error` torrent whose data is still in place.
Caveat: rechecking a torrent whose files were legitimately deleted will make it
re-download them.

### core.get_path_size(path) -> int
Returns byte size, or **-1 if the path is inaccessible**. Deluge's docstring:
"non-existent **or insufficient privileges**". Useful as an existence probe from
another container that has no mount of Deluge's filesystem — but a permissions
problem is indistinguishable from a missing file.

### core.get_torrents_status(filter_dict, keys)
Relevant keys: `name`, `state`, `message`, `save_path`, `label`, `progress`,
`total_done`, `total_size`, `seeding_time`, `files`.
- `state` values seen live: `Seeding`, `Queued`, `Downloading`, `Paused`,
  `Checking`, `Moving`, `Error`.
- `state == 'Error'` covers torrent/file errors; the detail string is in
  `message` (`statusmsg`). **Tracker** problems do NOT set Error — they surface
  in `tracker_status` separately.

Other used: `core.remove_torrent(hash, remove_data)`, `core.remove_torrents([hashes], remove_data)`,
`core.queue_top/queue_bottom([hashes])`, `label.get_labels`, `label.add`, `label.set_torrent`.

## Seed/peer status fields (confirmed against deluge-torrent/deluge source, `deluge/core/torrent.py`, 2026-08-13)

Deluge exposes both a swarm-wide (tracker-reported) count and a locally-connected count for seeds and peers — easy to grab the wrong one:

| Field | Meaning | libtorrent source |
|---|---|---|
| `total_seeds` | Swarm-wide seed count as reported by the tracker (includes yourself while actively seeding) | `status.num_complete` |
| `num_seeds` | Seeds we are actually connected to as peers (our own peer connections only) | `status.num_seeds` |
| `total_peers` | Swarm-wide peer/leecher count (tracker-reported) | `status.num_incomplete` |
| `num_peers` | Non-seed peers we're connected to | `status.num_peers - status.num_seeds` |

**Gotcha:** `total_seeds` (and `total_peers`) can be `-1` when the tracker hasn't scraped yet — this means "unknown," not "zero." Code that gates on swarm seed count must treat `-1`/missing as "don't know, don't touch," never as "0 is safe."

### Where used (plex-automation)

`plex-automation/arr-webhook.py`:
- `get_all_torrents()` (~line 292) requests `['name', 'label', 'save_path', 'seeding_time', 'progress', 'state', 'total_uploaded', 'total_seeds']` via `core.get_torrents_status`.
- `cleanup_stalled_seeds()` uses `total_seeds` as a swarm-safety gate before unseeding a stale/low-upload torrent — only unseeds if `total_seeds > STALL_MIN_SWARM_SEEDS` (env, default 5). Missing/negative `total_seeds` blocks removal.
- `remove_torrent(torrent_hash, remove_data=True)` — `remove_data=False` unseeds only (Deluge stops tracking/seeding, files stay on disk); default `True` deletes files too. The stalled-seed cleanup path always passes `remove_data=False` since it targets library-seed torrents where the seeded file is the live Plex media.

## Path topology (IMPORTANT — the two containers disagree)

Deluge's namespace maps the Unraid share at `/data`; `arr-webhook` (<lan-ip>)
mounts pieces of the same share at different paths:

| Deluge sees | arr-webhook sees |
|---|---|
| `/data/Downloads` | `/data/Downloads` (identity — only after the mount added 2026-08-06) |
| `/data/Media/Movies` | `/media/movies` |
| `/data/Media/TV Shows` | `/media/tv` |
| `/data/Media/Music` | `/media/music` (ro) |
| `/data/Media/Adult` | *not mounted* |

Any path crossing the boundary must be translated (`DELUGE_PATH_MAP` in
arr-webhook.py). Verify a translated path with `core.get_path_size` before
handing it to `move_storage`.

### Torrents seed from the media library, not just from Downloads
Live sample of ~1569 torrents (2026-08-06) — `save_path` distribution:
```
317 /data/Downloads/Incomplete        35 /data/Media/TV Shows/<series>/Season NN
309 /data/Media/Adult                 28 /data/Media/TV Shows/South Park
245 /data/Downloads/Complete/sonarr   ...many more /data/Media/TV Shows/...
106 /data/Downloads/Complete/radarr
101 /data/Downloads/Just4Seeding
 38 /data/Downloads/Complete/lidarr
```
Consequence: a large fraction of torrents seed the **library files themselves**.
A wrong `move_storage` can therefore move real library media, not just download
scratch. This is why auto-relink is opt-in (`DELUGE_RELINK=0` default).

Data layout: `<save_path>/<torrent name>` for multi-file torrents; for
single-file torrents `name` includes the extension, so `join(save_path, name)`
is the correct existence probe in both cases.
