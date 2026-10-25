# Darkbloom + BloomGauge Beta

Private operational reference (LAN endpoints, local paths, and our own usage). Verified 2026-10-06 on the Mac Studio (Apple M4 Max, 64 GB, 40 GPU cores, macOS 27), `darkbloom 0.9.17`, from live read-only `darkbloom status` / `models list --all` / `models catalog`, the local collector API, `provider.toml`, machine-config `docs/bloom-idle-switch.md` + `docs/darkbloom-keepwarm.md` (commit 048ee71), and the 2026-10 dispatch log. **Never write the local API key or machine ID into notes.**

Darkbloom is a provider network: the Swift-native `darkbloom` CLI turns the Mac into an MLX inference provider that connects to the coordinator (`wss://api.darkbloom.dev/ws/provider`, App Attest authorization) and serves **public network requests** for earnings. We also use the same running provider as the **local inference backend** for the dispatch queue (since 2026-10-01; Ollama/llama-server are being retired). BloomGauge Beta is a separate macOS app (collector + dashboard) that watches the provider, its earnings and demand, and can switch/pin models ("optimizer").

## Surfaces

| Surface | Address | Auth | Used by |
|---------|---------|------|---------|
| Local OpenAI-compatible endpoint (direct mode) | `http://127.0.0.1:8000/v1` (`~/.darkbloom/local.json`: base_url, port, pid, api_key, version) | local API key from `darkbloom local` | dispatch queue / workers (`~/bin/darkbloom_chat.py`) |
| Provider backend | port 8100 (internal) | none exposed | the provider |
| Coordinator | `wss://api.darkbloom.dev/ws/provider` | App Attest (macOS 27+) | the provider |
| BloomGauge collector API | `http://127.0.0.1:8765` | none, but POSTs need header `X-Bloom-Action: optimizer` | BloomGauge UI, `bloom-idle-switch.py` |

The local endpoint **requires the local API key**: even `GET /v1/models` answers 401 without it (so a health check that omitted the key was a bug in the first widen script). Get the key with `darkbloom local` (prints endpoint and key; do not paste it anywhere). Darkbloom's public docs (docs.darkbloom.dev) do not document local mode, structured output, error shapes or concurrency; what is known comes from probing (see queue notes below).

## CLI (`~/.darkbloom/bin/darkbloom`)

| Command | What it does / gotchas |
|---------|------------------------|
| `start --model A --model B [--local-endpoint]` | Starts the provider as a background (launchd) service. **Rewrites `enabled_models` in `provider.toml`** to the given set and restarts the service; this is the only way to change the filter. |
| `switch --model X` | Changes hosted models **without restart**, but **rejects any model outside the enabled filter** ("missing or unsupported local model(s)") and changes nothing, even for our own models when listed alongside others. No-restart narrowing/widening by editing `provider.toml` then `switch` is untested live. |
| `stop` / `restart` | `stop` drains accepted requests then persistently stops; `restart` keeps the current selection. |
| `status` | Config, hardware, filter, warm models, "Live load memory: N GB usable now (no eviction)", most-recently-used model, slot posture per model (kv=paged, mtp), serving concurrency caps, autopilot state, daemon pid/uptime. |
| `models list` / `models list --all` | `list` is **filtered by `enabled_models`**; `--all` shows every locally downloaded model (ID, type, quant, size, EST MEM). |
| `models catalog` | Supported-model list with a check mark for eligible+installed, RAM floor per model and "not eligible on this machine (missing: ...)". |
| `local` | Shows the local endpoint and API key (secret). |
| `idle` | Chooses what happens to loaded models when no requests arrive. Here: "always ready (models stay loaded)" (`idle_timeout_mins = 0`). |
| `beta` | Configurable beta features (here: `gemma-prefill-layer18=on`, `gemma-weighted-r1=on`, `mtp=auto`). |
| others | `doctor`, `verify`, `login`/`logout`, `enroll`/`unenroll` (App Attest, MDM), `benchmark`, `logs`, `report`, `update`, `autoupdate`. |

`autoupdate` is deliberately kept on (do not pin); updates restart the provider, and the worker/queue waits on `/health` and resumes.

## `~/.config/darkbloom/provider.toml` (keys that matter)

| Key | Current value | Notes |
|-----|---------------|-------|
| `[backend] enabled_models` | `['qwen3.6-35b-a3b-vl-mtp-mxfp8', 'Qwen3.5-9B']` | The **advertised/served filter**. Anything not listed is not served (and `switch` cannot select it). |
| `preload_models` | `['qwen3.6-35b-a3b-vl-mtp-mxfp8', 'qwen3.5-9b']` | Loaded at startup; `startup_preload = true`, `startup_preload_timeout_secs = 120`. Startup preload never evicts a resident model ("Preload skipped (no eviction)" when it would have to). Note the lowercase `qwen3.5-9b` id here vs `Qwen3.5-9B` in the filter. |
| `max_model_slots` | `2` | Max resident models. |
| `idle_timeout_mins` | `0` | Disables time-based unloading only; does **not** stop request-driven eviction. |
| `[provider] memory_reserve_gb` | `4` | Config value; the load-budget math actually applied by BloomGauge and the provider uses ~6.4-6.5 GB (below). |
| `[backend.model_autopilot]` | autopilot off | Autopilot (`pinned_models`) is the only real eviction pin; it is off, and BloomGauge refuses to manage an Autopilot-enrolled provider. |
| `[backend.engine_v2_max_concurrent_by_model]` | 9B = 2, 35B = 4 | Serving concurrency caps (default operator cap 4; wider needs an exact reviewed model/runtime/hardware profile). |

**`start` rewrites the filter, and provider restarts can reset it.** On 2026-10-06 the provider was drained/restarted at 12:40:42 and 13:03:03 PDT and `enabled_models` came back as `[gemma-4-26b-qat-4bit]` only (somebody ran a gemma-only `start`; unidentified). The 35B stopped being served, queue jobs failed instantly with "darkbloom not serving", and the weights were intact in `/Volumes/NVMe-Models/hf-cache`. Restored with `darkbloom start --model qwen3.6-35b-a3b-vl-mtp-mxfp8 --model Qwen3.5-9B --local-endpoint`. The queue now treats "provider not serving" as an infra wait (no failure, no attempt burn) and raises a DARKBLOOM-DRIFT alert (60 s check) with the fix command.

## Models: catalog and fit (`models catalog` + `models list --all`)

| Model id | Size on disk | EST MEM | Catalog RAM floor | Status on this Mac |
|----------|--------------|---------|-------------------|--------------------|
| `Qwen3.5-9B` | 5.7 GB | 6.8 GB | 24 GB | installed, **our pair** |
| `qwen3.6-35b-a3b-vl-mtp-mxfp8` | 19.8 GB | 23.8 GB | 32 GB | installed, **our pair** (queue default model) |
| `ternary-bonsai-2-27b` | 8.0 GB | 9.6 GB | 24 GB | installed |
| `gemma-4-26b-qat-4bit` | 14.5 GB | 17.4 GB | 36 GB | installed |
| `qwen3.5-35b-a3b` | 19.4 GB | 23.3 GB | 36 GB | installed |
| `nvidia-nemotron-3.5-lightning` | 17.3 GB | 20.7 GB | 48 GB | installed |
| `gemma-4-26b` / `gemma-4-26b-8bit` (rollback) | 26.0 GB each | 31.2 GB each | 36 / 64 GB | installed; excluded from the wide set |
| `gpt-oss-20b` | (~12.1 GB) | n/a | 24 GB | eligible but **not downloaded** (low pay; excluded) |
| `EigenLabs/Qwen3.8-27B-4bit-mtp` | (~16.3 GB) | n/a | 36 GB | **not eligible: needs Apple M5 + NAX runtime** (missing `apple_m5`, `mlx_nax`) |
| `mimo-v2.6-flash-mopd` (Xiaomi MiMo-V2.6-Flash) | (~172.9 GB) | n/a | 256 GB | far too big for 64 GB |

(The old `qwen3.8:27b-q4_K_M` Ollama tag is not runnable on Darkbloom; legacy tags are mapped to the Darkbloom model for display only.)

### Memory math

* A cold load needs **EST MEM + ~6.4-6.5 GB reserve** (BloomGauge `loadBudget.reserveGB` = 6.4; dispatch notes cite 6.5). Check: qwen3.6 23.8 + 6.5 = **30.3 GB**, exactly the "Insufficient memory (need 30.3GB)" 429 the provider returned when a cold 35B load was refused while idle Gemma was still resident.
* Our pair (provider ~29.5 GB resident incl. ~1.9 GB cache) leaves roughly **12 GB usable** (status showed 12.1 GB on 2026-10-06 13:5x; BloomGauge `memory.availableGB` 12.2). "Live load memory" fluctuates with file cache (it read 5.2 GB and 25.5 GB at other times), so the queue's fit routing keys off model residency, not that number (a stale `darkbloom-usable-peak.json` once caused wrong overflow to Unraid).
* `darkbloom status` prints a **60 GB nominal hardware budget** (not live free RAM). The brief for this note cited a ~51 GB Metal working set; that figure was **not found in any local source or log and is unverified**. Treat usable memory as: working set minus resident weights minus reserve.
* A third model rarely fits beside the pair: by memory alone the smallest (bonsai, 9.6 GB) might, but the binding rule is `max_model_slots = 2` plus memory, so raising slots to 3 does not help.

## Eviction behaviour (read-only analysis, 2026-10-06)

* With `max_model_slots = 2`, a request for an **enabled but non-resident model lazy-loads it and evicts the least-recently-used idle resident model** (a Gemma queue job lazy-loaded in ~12 s and evicted the idle 35B on an earlier test).
* **Only models with an in-flight request are protected**, plus an Autopilot pin, plus an enabled set that fits in the slots. `idle_timeout_mins = 0` and startup preload give no later protection. Nothing in `provider.toml` pins a model.
* So a wide `enabled_models` means **public demand can evict a queue model**. Evidence 2026-10-06: wide filter (6 models) applied 13:54:51; qwen3.6 loaded 13:55:07 and the 9B 13:55:37 (startup preload); at **13:56:35** a network-routed request for `nvidia-nemotron-3.5-lightning` started a cold load (weights hash 13:56:35, loaded 13:56:51, 16 s) and **qwen3.6 (LRU idle, 23.8 GB) was evicted at 13:56:37**. BloomGauge's history DB shows no switch/probe/warm event in that window (`opt_events`, `demand_switch_runs`, `model_switch_alerts` empty; optimizer mode `demand`, phase `waiting`, no pin), and the network table showed nemotron actively served by ~3-4 jobs across ~150 routable providers. An earlier note guessed BloomGauge's warmup (which was targeting nemotron) was the cause; the history-DB check rules that out. The set was rolled back to the pair at 21:10Z (14:10 PDT).
* Chosen design (see below): advertise only what fits (set == pair) while the queue works; let BloomGauge's optimizer roam only when idle.

## Earnings

About **$0.003 per hour** for this Mac at the time of the 2026-10-06 analysis, so every rule favours the queue over chasing earnings (BloomGauge's own earnings fetch also logged one failure on 2026-10-05). `Requests served` and tokens appear in `darkbloom status`.

## BloomGauge Beta

App `/Applications/BloomGauge Beta.app` (collector in its `Contents/Resources/collector.py`). Data and logs: `~/Library/Application Support/Bloom Dashboard Beta/` (`history.sqlite3` with `opt_events`, `demand_switch_runs`, `model_switch_alerts`; `network-view-v1.json`; **log `logs/bloom.log`**; `collector.lock`). It also reads Darkbloom's `~/.darkbloom/loaded-models.json` and `/health`.

### Collector API (`http://127.0.0.1:8765`)

| Call | Purpose |
|------|---------|
| `GET /api/optimizer` | Optimizer state: `mode` (here `demand`), `status` (`optimizing`), `detail` (**"Holding your pick Qwen3.5-9B + qwen3.6-35b-a3b-vl-mtp-mxfp8. BloomGauge will not ..."** when our pair is pinned), `currentModels`, `models[]` (per model: `available`, `downloaded`, `memoryGB`, `loadBudget{requiredGB, afterUnloadGB, reserveGB}`, `evidence`, `demand`), `demandAuto.manager{pinned, home, homeNotice}`, `resumeDemand.hasSavedPlan`, `memory{availableGB, providerGB, ...}`, `warmup`, `switchRules`, `policy` (min 6 h/2 days/20 jobs, 20% improvement), `lastSwitchResult`. |
| `GET /api/optimizer/control` | Returns `controlVersion` (an opaque hash), `providerVersion`, `currentModel` (e.g. `@combo:["Qwen3.5-9B","qwen3.6-35b-a3b-vl-mtp-mxfp8"]`), `actualMode`, `hasSavedPlan`, `selected[]`. |
| `GET /api/model-control` | Model switch state incl. `warmup.status`, per-model `canStart`/`loadBudget`. |
| `POST /api/optimizer/control` | Control actions. Header **`X-Bloom-Action: optimizer`** required. Body `{"action": ..., "requestId": "<uuid>", "expectedControl": "<controlVersion>"}`. |

Control actions used by `bloom-idle-switch.py`:

* **`release-pin`** — hand the choice back to BloomGauge (back to automatic/demand mode). The caller must **re-read `/api/optimizer` and only treat it as released when `demandAuto.manager.pinned` is false**; a still-pinned pick is a stuck pick to retry.
* **`keep-current`** with `"model": "@combo:[...]"` — cancel a planned switch and re-pin the current pair without a reload (used only when a `homeNotice` exists).
* `resume-demand` (an `/api/optimizer` action, different thing) re-applies a saved demand plan.

BloomGauge's API cannot select a *pair*: a pair started outside it with `darkbloom start --model A --model B --local-endpoint` is **adopted as an external pin**, which is what shows as "Holding your pick" (the demand-mode text is the way to verify a restore). BloomGauge's own probes/warmups are recorded as stall-probe / switch-probe events.

## Our usage

* Provider serves the pair (`Qwen3.5-9B` + `qwen3.6-35b-a3b-vl-mtp-mxfp8`, preloaded) as the dispatch queue's backend at `127.0.0.1:8000`; ~2.4 s to 37.7 s per answer from 3.5k to 56k prompt tokens (needle retrieval correct at 3.5k/7k/14k/28k/56k with `chat_template_kwargs.enable_thinking=false`; `/v1/models` exposes no context length). Stage medians after the 2026-10-01 migration: auto-author ~4.5 min, refine ~1.6 min, slice coding ~0.9 min.
* Queue behaviour: `/v1/models` lists hosted-but-cold models; a cold-load refusal (429 insufficient memory) is waited on without charging an attempt; `darkbloom_chat.py` walks a schema ladder (strict, lenient, relaxed) for structured output.
* Opt-in helpers in machine-config (`bin/`, mirrored to `~/bin`), **all default OFF and not enabled as of 2026-10-06 14:2x** (no `.enabled` kill-switch files under `~/.ollama-dispatch/`; plist drafts in `~/.ollama-dispatch/proposals/`; queue-daemon restart needed for the queue-side holds):
  * `bloom-idle-switch.py` — work/idle lifecycle: when the queue has been idle for `idle_minutes` (10) release the pin so the optimizer may chase the best-paying model; when work arrives (or a bundle is committed) restore the pair via `darkbloom start` and verify provider.toml + `/health` + `/v1/models` + `loaded-models.json` + BloomGauge warmup + the "Holding your pick" text; queue infra-waits (`bloom_idle_hold()`) while restoring. Flap protection: 10 min idle persistence, 30 min cooldown, max 3 aways/day; restore ignores cooldown and cap. Optional `~/.ollama-dispatch/bloom-idle-switch.json` (`idle_minutes`, `cooldown_minutes`, `max_away_per_day`, `pair`, `restore_wait_s`, `switch_wait_s`). Log `~/.ollama-dispatch/bloom-idle-switch.log`; state `bloom-idle-state.json`; status `bloom-idle-switch.py --status`.
  * `darkbloom-keepwarm.py` — 60 s guard: if a queue model is not resident it alerts, touches the resident queue models first (so the other is never the LRU victim) and sends a 1-token request to the missing one; stands down while idle-switch is away/restoring; never restarts Darkbloom or edits provider.toml; queue side `darkbloom_warm_hold()` infra-waits jobs for a held model.
  * `darkbloom-apply-wide-models.sh` — dry-run by default; `--apply` refuses while a job runs, snapshots provider.toml, runs `darkbloom start`, verifies filter + both queue models, rolls back on failure; `--rollback`.
  * `darkbloom-widen-now.sh` — the one-off drain-and-widen that produced the 2026-10-06 eviction evidence (wide set was qwen3.6, 9B, bonsai-2-27b, nemotron-3.5-lightning, qwen3.5-35b-a3b, gemma-4-26b-qat-4bit).
  * Enable: `touch ~/.ollama-dispatch/bloom-idle-switch.enabled` (and `darkbloom-keepwarm.enabled`), install the launchd plist(s), restart the queue daemon at idle. Limits: the away release rarely fires while pending work or a committed bundle exists, and each idle-to-work edge costs reload minutes.

## Known gaps / contradictions in the sources

* `darkbloom-keepwarm.md` opens by saying it *supersedes* the idle-away design in `bloom-idle-switch.md` ("kept, unloaded"), but its own evidence section and the newer lifecycle section of `bloom-idle-switch.md` say the work/idle lifecycle **is** the chosen design with keep-warm as insurance. Treat the lifecycle design as current; the header is stale.
* `memory_reserve_gb = 4` in `provider.toml` vs ~6.4-6.5 GB reserve in the actual load math; the config value's effect is unverified.
* The ~51 GB Metal working set is unverified (see memory math).
* `darkbloom switch` narrowing without a restart is untested live.
