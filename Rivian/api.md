# Rivian API (Unofficial)

**Status:** Unofficial — reverse engineered from iOS app traffic. May break without notice.  
**GraphQL Gateway:** `https://rivian.com/api/gql/gateway/graphql`  
**Charging API:** `https://rivian.com/api/gql/chrg/user/graphql`  
**WebSocket:** `wss://api.rivian.com/gql-consumer-subscriptions/graphql`

---

## Authentication

**Flow:** CSRF token → Login (email/password) → [OTP if MFA] → tokens saved

### Required Base Headers
```
User-Agent: RivianApp/707 CFNetwork/1237 Darwin/20.4.0
Accept: application/json
Content-Type: application/json
Apollographql-Client-Name: com.rivian.ios.consumer-apollo-ios
```

### Step 1 — Create CSRF Token
```graphql
mutation CreateCSRFToken {
  createCsrfToken {
    csrfToken
    appSessionToken
  }
}
```
No auth headers needed. Returns `csrfToken` and `appSessionToken`.

### Step 2 — Login
```graphql
mutation Login($email: String!, $password: String!) {
  login(email: $email, password: $password) {
    __typename
    ... on MobileLoginResponse {
      accessToken
      refreshToken
      userSessionToken
    }
    ... on MobileMFALoginResponse {
      otpToken
    }
  }
}
```
**Additional headers:** `Csrf-Token: <csrfToken>`, `A-Sess: <appSessionToken>`

- If response is `MobileLoginResponse` → auth complete, save tokens
- If response is `MobileMFALoginResponse` → save `otpToken`, proceed to Step 3

### Step 3 — Login with OTP (if MFA required)
```graphql
mutation LoginWithOTP($email: String!, $otpToken: String!, $otpCode: String!) {
  loginWithOTP(email: $email, otpToken: $otpToken, otpCode: $otpCode) {
    accessToken
    refreshToken
    userSessionToken
  }
}
```
Same headers as Step 2.

### Tokens
After login, include in ALL subsequent requests:
- `Csrf-Token: <csrfToken>`
- `A-Sess: <appSessionToken>`
- `U-Sess: <userSessionToken>`

No explicit refresh token mutation observed — tokens may be long-lived or require re-login.

---

## User / Vehicles

### Get Current User + Vehicles
```graphql
query GetCurrentUser {
  currentUser {
    id
    firstName
    lastName
    email
    vehicles {
      id
      name
      vin
      vehicle {
        id
        vin
        make
        model
        modelYear
      }
    }
  }
}
```
`vehicles[0].id` is the vehicle ID used for state queries.

---

## Vehicle State

### Query Vehicle State
```graphql
query GetVehicleState($vehicleID: String!) {
  vehicleState(id: $vehicleID) {
    cloudConnection { lastSync isOnline }
    batteryLevel { timeStamp value }        # 0-100 percent
    distanceToEmpty { timeStamp value }     # KILOMETERS — divide by 1.609344 for miles
    batteryLimit { timeStamp value }        # 0-100 percent
    timeToEndOfCharge { timeStamp value }   # minutes
    chargerState { timeStamp value }        # see values below
    chargerStatus { timeStamp value }
    powerState { timeStamp value }
    vehicleMileage { timeStamp value }      # METERS — divide by 1609.344 for miles
    doorFrontLeftLocked { timeStamp value } # "locked" | "unlocked"
    doorFrontRightLocked { timeStamp value }
    cabinPreconditioningStatus { timeStamp value } # see values below
    chargePortState { timeStamp value }
    chargerDerateStatus { timeStamp value }         # active throttling reason
  }
}
```

**chargerState values (actively charging states only — do NOT use .includes('charging')):**
- `disconnected` — not plugged in
- `not_charging` — plugged in, not charging
- `charging_ready` — plugged in, ready but not charging (contains "charging" — is NOT active)
- `charging` — actively charging
- `charging_active` — actively charging (alternate value)
- `charge_starting` — contactor closing
- `charging_ac_1ph` — charging single-phase AC
- `charging_ac_3ph` — charging three-phase AC
- `charge_complete` — fully charged

**chargerDerateStatus values:**
- Empty string / `no_derate` / `none` / `inactive` / `normal` — not throttled
- Non-empty other value — vehicle is being throttled. Specific strings aren't
  publicly documented; capture the raw value when it occurs. Reasons commonly
  include thermal, battery temperature, voltage issues. Source: `bretterer/home-assistant-rivian`
  exposes the raw value as a sensor without an enum mapping.

**cabinPreconditioningStatus values:**
- `system_idle` — climate off
- `not_available` — no phone key / unavailable (NOT the same as on)
- `cooling` — AC cooling
- `heating` — heating
- `defrost` — defrost
- `ventilation` — fan only
- `preconditioning` — preconditioning

**Unit notes:**
- `distanceToEmpty` — **kilometers** (NOT miles; confirmed empirically 2026-08-30: raw 255 at 55% SOC = 287mi pack, and live dashboard raw 146 = 145km = 90mi in the Rivian app) — divide by **1.609344** for miles. The open web does not document this unit anywhere (a verified-research pass over RivDocs, rivian-python-client, the GraphQL Postman collection and community SDKs found no statement of it); determined from live vehicle data.
- `vehicleMileage` — **meters** (confirmed: ~27k mi shows as ~43M raw) — divide by 1609.344
- All other numeric fields — native units as labeled

---

## Vehicle Commands

Commands require HMAC signing with:
- Vehicle public key (`vas.vehiclePublicKey` from `currentUser`)
- Phone private key (enrolled phone)
- `vasPhoneId`, `deviceId`, `vehicleId`

This makes commands complex to implement without the BLE key pair. The HA integration (`bretterer/home-assistant-rivian`) has a working implementation for reference.

**For the dashboard, vehicle state is read-only until command signing is implemented.**

---

## Full vehicleState field list (RivDocs)

Fields available beyond the ones documented above (source: [RivDocs GetVehicleState](https://rivian-api.kaedenb.org/app/vehicle-info/vehicle-state/)):

**Location (GNSS):** `gnssLocation { latitude longitude timeStamp }`, `gnssSpeed { value }`, `gnssAltitude { value }`, `gnssError { positionHorizontal positionVertical speed bearing }`

**Battery / charging:** `batteryLevel`, `batteryLimit`, `batteryCapacity`, `batteryHvThermalEvent`, `batteryHvThermalEventPropagation`, `chargerState`, `chargerStatus`, `chargerDerateStatus`, `remoteChargingAvailable`, `timeToEndOfCharge`, `distanceToEmpty`, `rangeThreshold`, `chargePortState`

**Doors / windows / closures:** `doorFrontLeftLocked/Closed`, `doorFrontRightLocked/Closed`, `doorRearLeftLocked/Closed`, `doorRearRightLocked/Closed`, `windowFrontLeft/RightClosed`, `windowRearLeft/RightClosed`, plus corresponding `*Calibrated` fields, `closureFrunkLocked/Closed/NextAction`, `closureLiftgateLocked/Closed/NextAction`, `closureSideBinLeft/Right...`, `closureTailgateLocked/Closed/NextAction`, `closureTonneauLocked/Closed`, `gearGuardLocked`

**Climate:** `cabinClimateInteriorTemperature`, `cabinClimateDriverTemperature`, `cabinPreconditioningStatus`, `cabinPreconditioningType`, `petModeStatus`, `petModeTemperatureStatus`, `defrostDefogStatus`, `steeringWheelHeat`, `seatFrontLeft/RightHeat`, `seatRearLeft/RightHeat`, `seatFrontLeft/RightVent`

**Vehicle:** `powerState`, `gearStatus`, `vehicleMileage`, `alarmSoundStatus`, `wiperFluidState`, `brakeFluidLow`, `driveMode`, `serviceMode`

**Tires:** `tirePressureStatusFrontLeft/Right`, `tirePressureStatusRearLeft/Right`, plus corresponding `tirePressureStatusValid*` flags

**OTA:** `otaCurrentVersionNumber/Week/Year/GitHash`, `otaAvailableVersionNumber/Week/Year/GitHash`, `otaDownloadProgress`, `otaInstallDuration`, `otaInstallProgress`, `otaInstallReady`, `otaInstallTime`, `otaInstallType`, `otaStatus`, `otaCurrentStatus`

**Gear Guard:** `gearGuardVideoStatus`, `gearGuardVideoMode`, `gearGuardVideoTermsAccepted`

**Not exposed:** 12V battery state — no publicly documented field.

---

## Charging derate reasons

`chargerDerateStatus.value` returns a raw string when active. Documented no-op values: empty string, `no_derate`, `none`, `inactive`, `normal`. Any other value = active throttling. **Reason strings are not documented publicly** — capture the raw value when it fires. Community expectation: thermal reasons (handle temperature, battery temp, cable) and voltage-related reasons. Verified by `bretterer/home-assistant-rivian` which exposes the field as a raw string sensor with no enum map.

Similarly, `batteryHvThermalEvent` and `batteryHvThermalEventPropagation` return non-empty strings during HV battery thermal excursions. Values undocumented; capture raw.

---

#### Live charging power/current — separate GraphQL service, schema drifted (2026-08-08)

`vehicleState` (main `gateway/graphql`) has NO power/current field — confirmed
by field-name probing that this endpoint gives zero "did you mean" hints on
invalid guesses (error masking, standard prod hardening), so it's a dead end
for undocumented-field discovery.

Rivian's app gets live charging power/current from a genuinely separate
GraphQL service: `https://rivian.com/api/gql/chrg/user/graphql`. Real,
live-tested against an active home AC charging session (not guessed):

- **`getLiveSessionData(vehicleId: ID!)`** — the obvious query from
  `github.com/bretterer/rivian-python-client`'s `schemas/charging.graphql`
  (`current`, `power`, `soc`, `totalChargedEnergy` fields) — **stale, does
  not exist on the live server.** Error: "Cannot query field
  getLiveSessionData... Did you mean getLiveSessionHistory or
  getSessionStatus?" That community client's schema file has drifted from
  Rivian's real current API.
- **`getNonRivianUserSession`** (no args) — also in the old schema,
  specifically documented for "current non-Rivian charging session" (the
  exact home-AC case) — **also gone, confirmed live.**
- **`getSessionStatus`** — exists, but takes `jobId: ID!` not `vehicleId`.
  Almost certainly an async command-job status poll (e.g. start/stop
  charging command completion), not charging telemetry. Not pursued.
- **`getLiveSessionHistory(vehicleId: ID!)`** — the one query that
  validates. Real confirmed field names: `startTime`, `vehicleId`,
  `chargerId`, `transactionId`, `current { value updatedAt }`, `chartData`
  (items are type `ChartDataPoint`, NOT `FloatValueRecord` like the old
  schema claimed — exact `ChartDataPoint` shape not yet probed). **But
  every field came back `null` and `chartData` came back `[]` during a
  real, active home AC charging session.** Working theory: this whole
  `chrg` surface is scoped to Rivian's own charging network (DC fast /
  Adventure Network, real transaction IDs) and structurally has nothing for
  a private, unregistered Level 2 charger. Untested against a real Rivian
  DC fast-charge session — that's the next thing to try if one happens.

**Auth for the charging endpoint**: same three headers as the main gateway
(`Csrf-Token`/`A-Sess`/`U-Sess`) work fine — `bretterer`'s client uses only
`U-Sess` for this specific call but sending all three didn't break anything.

**The real remaining lead for home-AC live power**: Rivian's newer
"Parallax" system (`rivian-api.kaedenb.org/app/parallax/domains/charging/`)
— live session state moved to Protobuf-encoded "RVM" objects
(`charging.session.status`, etc.), and live power/current specifically
looks like it's under `energy_edge_compute.graphs.charging_graph_global` —
a websocket subscription with binary payloads, structurally different from
every plain-GraphQL approach above. `bretterer/rivian-python-client` has
unexplored `parallax.py` and `ws_monitor.py` files that may already
implement this — check those before reverse-engineering from scratch.
Deferred in `~/Desktop/GitHub Projects/TODO.md` until weekly usage limits
reset.

**Lesson for next time**: this community client's schema file
(`charging.graphql`) is NOT a reliable source of truth — it's drifted
significantly from Rivian's real live API. The `chrg` endpoint's GraphQL
validation errors DO leak useful "did you mean" field-name suggestions
(unlike the main `gateway` endpoint), which is how the real current field
names got found — worth using that technique again for future schema
exploration on this specific endpoint.

## WebSocket subscriptions

Real-time push endpoint (avoids polling entirely):
`wss://api.rivian.com/gql-consumer-subscriptions/graphql`

Use standard GraphQL-over-WebSocket protocol. Requires the same `Csrf-Token` / `A-Sess` / `U-Sess` headers as GraphQL POST. Subscription operations mirror `vehicleState` query shape. Community projects (`bretterer/*`) implement this for near-realtime updates. Not currently used by ev-dashboard; polling with backoff is sufficient.

---

## Rate limiting

Rivian does not publish rate limits. Throttling responses are **opaque** — errors do not indicate whether they're throttling or genuine failure. Community-tested backoff (from `bretterer`):

> "I use exponential backoffs to the point where I would stop seeing errors. The errors are really really opaque and don't tell you if it's throttling or not. I do 15, 30, 60, 120, 240 minutes."

ev-dashboard uses: 15 → 30 → 60 → 120 → 240 min on consecutive `fetchRivianVehicleState` errors, reset on first success. Interactive command calls are not backed off — the user is present and will retry manually if needed.

---

## Session lifetime

No refresh mutation is documented. Tokens appear to work for ~90 days before failing with 401. ev-dashboard treats the token's `savedAt` timestamp as the session start and:

1. Sets a `rivian_reauth_due_soon` flag at day 83 (7 days before assumed expiry).
2. Sets `rivian_reauth_required` at day 90.
3. Also sets `rivian_reauth_required` on any 401 from the gateway, regardless of age.

User must re-run the login flow (email + password + OTP) to refresh. Consider surfacing a "Sign in to Rivian" admin route ahead of day 90.

---

## Notes
- No public/official API — reverse engineered from iOS app (v707)
- `apollographql-client-name` header is required or requests are rejected
- Tokens appear session-based; ~90-day lifetime observed; no documented refresh
- Source references: `bretterer/rivian-python-client`, `bretterer/home-assistant-rivian`, [`rivian-api.kaedenb.org`](https://rivian-api.kaedenb.org/)

## Gotchas confirmed 2026-08-06 (ev-dashboard reauth/charging bug fix)

- **`chargerState.timeStamp` is a last-*changed* stamp, not a last-seen/heartbeat stamp.** A steady multi-hour charge session doesn't update it, so using its age as a staleness/validity gate is wrong — it ages past any reasonable threshold (15 min) while the vehicle is still actively, correctly charging. Use `isPluggedIn` (or `chargerStatus` directly) for validity, not `chargerState` age.
- **`chargerStatus` is the more reliable field for both plug and charge state** — treat it as authoritative when present; fall back to `chargerState` enum matching only as a secondary signal. Exact charging-state literal (e.g. `chrgr_sts_connected_charging`) is still **unconfirmed** from a primary source as of this note — `chrgr_sts_not_connected` is confirmed (from `bretterer/home-assistant-rivian`), the charging-side literal was recalled, not verified. Confirm against a live `[rivian] … chargerStatus="…"` log line during an actual charge session before hardcoding it anywhere.
- **Rivian's gateway sends raw JSON numbers on fields its own schema types as String** — already known for OTA version fields, now also confirmed for `chargerDerateStatus` (came back as a bare `0` instead of `"0"`). Any code calling `.trim()`/string methods on a Rivian API field should coerce with `String(...)` first rather than assuming the schema's stated type holds at runtime.

---

## Service / "In service" state (reverse-engineered from Android app 3.15.0, 2026-08-24)

There is **no `serviceMode` field on `vehicleState`** despite it appearing in the RivDocs field list
above — the polling vehicleState path shows NOTHING when the car is in service (all fields null,
`cloudConnection.isOnline=true` only). Service state lives in dedicated ops across TWO gateways.

### Detect "in service" — `GetAsyncMessageThreadList` / `CommsListDiscussions`
Endpoint: `https://rivian.com/api/gql/gateway/graphql` (the gateway ev-dashboard already uses).
`data.commsListDiscussions[]`, one per work order:
- `workOrderType`   — `SERVICE_CENTER` | `MOBILE_SERVICE`
- `workOrderStatus` — **`IN_PROGRESS`** (in service) | `DELIVERED` (done)
- `threadStatus`    — `WAITING_FOR_CUSTOMER` | `CLOSED`
- `workOrderId` (e.g. `WO-000002434913`), `appointmentDate`
**Rule:** in service iff any thread has `workOrderStatus == IN_PROGRESS`.

### Appointment/repair detail — `GetActiveRequests`
Endpoint: **`https://rivian.com/api/vs/gql-gateway`** — a SEPARATE vehicle-service gateway
ev-dashboard does NOT currently call (same session auth headers). `data.consumerServiceRequests.result[]`,
one per line item shown on the appointment screen:
- `title` (e.g. "Brake Fluid Flush", "Comprehensive Inspection", "SUV Cargo Area")
- `status` (e.g. `OPEN_SCHEDULED`) — drives the per-item ✓ / in-progress icon
- `concern` (free-text tech note), `requestType` (e.g. `ASSET_ISSUE`), `referenceId`, `createdAt`, `expiresAt`

### Work-order timing — `QueryByWorkOrderId` (aka `DetailsByWorkOrderId`), and `getAppointments`
- `appointmentStartAtIso` / `appointmentEndAtIso` — the DROP-OFF window (NOT a ready estimate)
- `visitStartAtIso` / `visitEndAtIso` — same window
- `completedAt` — **`null` while in progress**, real timestamp when done. This is the ONLY completion signal.

### THERE IS NO ESTIMATED-READY / ETA FIELD (verified 2026-08-24)
Pulled every datetime field from all five service ops and checked future-vs-now: **zero future-dated
fields.** Rivian does not expose an estimated completion time — the app itself only says "We'll reach
out with any updates." A dashboard can show status + line items + drop-off window + watch
`completedAt` null→timestamp, but must NOT present the drop-off window end as a "ready by" time
(it's in the past while the car is still being serviced). Don't re-hunt for an ETA field; it isn't there.

**Other authenticated gateway ops seen (mobile app):** GetVehicle, VehiclesAndEnrollments,
VehicleNames, getVehicleImages/getVehicleTireImages, CurrentUser, getRegisteredWallboxes,
vehicleOrders, GetServiceNotifications (`vehicleNotificationSignals.items`), SupportedFeatures,
chargingSites, GetReverseGeocode, getAppointments.

**Capture method:** Android 3.15.0 arm64 emulator on Apple Silicon + frida 16.7.3 + httptoolkit
unpinning WITHOUT native-tls-hook (Rivian is Java/OkHttp). Runbook: vault Homelab/android-emulator-macos.md.

### `GetActiveRequests` line-item `status` enum (confirmed 2026-08-24 via live refresh)
The per-item `status` on `consumerServiceRequests.result[]` advances as techs work, and is the
closest thing to a progress indicator Rivian exposes (there is still no ETA):
- `OPEN_SCHEDULED`        — not started yet
- `OPEN_IN_PROGRESS`      — being worked (spinner icon in the app)
- `CLOSED_WORK_COMPLETE`  — done (✓ checkmark in the app)
A dashboard can poll this and show a live "N/M complete" checklist. The overall `workOrderStatus`
stays `IN_PROGRESS` and `completedAt` stays null until ALL items finish.

### CRITICAL: service data is OWNER-SCOPED, invisible to shared/invited accounts (confirmed 2026-08-24)
Tested by logging the SAME app into both Penn's owner account and the ev-dashboard's shared account
(`dashboard@phd.simplelogin.com`, an invited user on the same R1S, same VIN <VIN>):

| op | owner account | dashboard (shared) account |
|---|---|---|
| CommsListDiscussions | 16 threads | **0** |
| GetAsyncMessageThreadList | 16 | **0** |
| GetActiveRequests (service line items) | 5 | **0** |

**Implication for ev-dashboard:** it logs in with the SHARED account, so ALL the service-mode
detection we mapped (`workOrderStatus==IN_PROGRESS`, the line-item checklist, `GetActiveRequests`)
returns EMPTY for it. "Add service queries to the dashboard" does NOT work with the current login.
The dashboard's only in-service signal via its shared account is the INDIRECT one: `vehicleState`
all-null while `cloudConnection.isOnline=true` (vehicle powered down on the service bus) — a
heuristic that can't distinguish in-service from deep-sleep. To get real service data the dashboard
would need the OWNER account's credentials for those specific ops (a security/credentials decision).

#### PENDING TEST: does a credentialed/keyed shared account gain service-data access?
The shared dashboard account currently has NO phone key on the vehicle. Penn will provision the
ev-dashboard account as a driver WITH A KEY next time he's in the R1S, then we re-capture to see if
service ops (CommsListDiscussions / GetActiveRequests / GetAsyncMessageThreadList) start returning
data for it. Hypothesis basis: the owner account's `GetVehicle.invitedUsers[]` carries `isCredentialed`
(bool) + `roles[]`, so access may be gated on being a credentialed/keyed user rather than strictly the
owner. If keying the account unlocks service data, the dashboard could use its own shared account for
in-service detection after all — no owner creds needed. UNRESOLVED until re-tested.

## WebSocket push subscriptions — CONFIRMED protocol (captured live 2026-08-24, both accounts)
Previously only the endpoint was known from community docs; now the full protocol + operations are
captured from the real app. This is the REAL-TIME alternative to ev-dashboard's current
polling-with-backoff (which is the root cause of the "car shows asleep / stale" problem — WARP
stalls + 15-240min backoff). Worth migrating the dashboard to.

**Endpoint:** `wss://api.rivian.com/gql-consumer-subscriptions/graphql` — graphql-ws protocol.
**Handshake:** client `connection_init` → server `connection_ack` → client `subscribe` (one per op) →
server streams `next` messages with the data → `complete`. Same session auth as the gateway
(Csrf-Token / A-Sess / U-Sess).

**Subscription operations observed (client→server `subscribe`):**
- `vehicleState($vehicleID: String!)` — the real-time vehicle state push. Query is
  `subscription vehicleState($vehicleID:String!){ vehicleState(id:$vehicleID){ __typename ...vehicleStateFields } }`
  with a large `vehicleStateFields` fragment (doors, charge, battery, gnss, etc. — same shape as the
  polled GetVehicleState). variables: `{"vehicleID":"01-245848997"}`.
- `vehicleCloudConnection`, `VehicleActiveDriverName`, `vehicleDepartureSchedules`,
  `VehicleConsentStatus`, `ParallaxMessages`.

**ACCESS — works for the SHARED account** (unlike service data): the ev-dashboard shared account
successfully subscribed to `vehicleState` and received `next` pushes (44 msgs). So the dashboard CAN
migrate to push with its existing shared login — no owner-creds problem here. (Sampled push data was
mostly null only because the vehicle is currently IN SERVICE; the mechanism is confirmed working.)

**Recommendation:** wire this as its own task, LIVE (not dormant), separate from the owner-scoped
service-mode feature. It replaces the poll loop in lib/rivian.ts with a persistent WS subscription +
reconnection. Verify full field population with a capture taken while the vehicle is OUT of service.

### STATUS: implemented in ev-dashboard (branch `rivian-websocket-state`, 2026-08-24)
Commit `bc0132e`, NOT merged/deployed. Files: `server/rivian-state-monitor.js` (persistent
graphql-ws sidecar, launched non-critical from `server/start.js`, writes
`keys/rivian-push-state.json`), `server/rivian-vehicle-state-fields.js` (field selection shared
by the polled query and the subscription so they cannot drift), `lib/rivian.ts`
(`mapRawVehicleState()` extracted so poll and push share one mapping;
`readRivianPushState()` / `rivianPushFresh()` / `getRivianVehicleState()`).

- Fallback: `getRivianVehicleState()` returns pushed state while `updatedAt` is < 15 min old,
  otherwise falls through to `fetchRivianVehicleState()` (backoff ladder unchanged). The poll is
  NOT deleted. `app/api/dashboard/route.ts` also skips its poll-interval cache gate while the
  push is fresh (a local file read costs no API quota).
- Pushed frames are treated as PATCHES, not snapshots: absent and null both mean "no news",
  never "cleared".
- Reconnect: exponential backoff to 5 min on close/error, plus a 10-min idle watchdog for a
  silently wedged socket (TCP open, no frames).
- STILL OPEN: full field population is unverified — the only live capture was taken while the
  car was IN SERVICE (nearly all fields null). Take an out-of-service capture and diff a pushed
  payload against a polled `GetVehicleState` response field-by-field before retiring the poll.


---

## Parallax enum mappings (empirically derived 2026-09-03)

**Source:** ev-dashboard `server/parallax-monitor.js` RVM decode, logged to `parallax-log/*.jsonl`,
pulled via `GET http://<lan-ip>:3000/api/admin/logs` (`parallaxLogFiles`). **4899 frames analyzed,
date range 2026-08-31T17:35Z → 2026-09-04T04:55Z**, spanning 4 home AC (L2) charge sessions on the R1S.
Ground truth = `powerKw` and, more reliably, the `totalChargedEnergyKwh` *rising vs flat* trend in time
order. Secondary source `bretterer/rivian-python-client` `parallax.py` decodes these three integers but
documents **no enum mapping** for them, so the mapping below is derived purely from the logs.

### CRITICAL caveat that shaped this analysis
`powerKw` and `totalChargedEnergyKwh` **freeze at their last value** when a session ends — the RVM stops
streaming and the monitor re-emits the last frame. In every session the post-charge tail held a constant
non-zero power for 30 min to **10 hours** (session 2026-09-03 held `powerKw=3.4` for 605 min while the
car was unplugged). **There is NOT a single true `powerKw==0` frame in the entire dataset.** Therefore
`powerKw>0` is NOT a safe "is charging" signal for Parallax — the enums flip correctly at end-of-session
while power goes stale. The true discriminator is `totalChargedEnergyKwh` *rising* (only 705 rising steps
exist, all in one state) vs *flat*.

### `chargingStateEnum`
| value | inferred meaning | frames | energy-rising steps | confidence |
|---|---|---|---|---|
| 3 | **Actively charging** | 4453 | 704 (the ONLY state that ever rises) | **HIGH** |
| 5 | **Charge complete / stopped** (plugged at target, then persists through unplug) | 359 | 0 (always flat) | MED (clearly "not charging"; "complete" label MED) |
| 1 | **Not charging / idle** (only ever seen as tail state 1/1/1, one session) | 87 | 0 (always flat) | LOW (rare, single session) |

### `plugConnectionStatus`
| value | inferred meaning | frames | context | confidence |
|---|---|---|---|---|
| 2 | **Plugged in / connected** | 4576 | all active-charge and at-target frames | **HIGH** |
| 1 | **Unplugged / disconnected** | 323 | only in frozen post-session tails; clean 2→1 flip at every session end, never reverts | **HIGH** |

### `displayStatus`
| value | inferred meaning | frames | context | confidence |
|---|---|---|---|---|
| 3 | **Charging** (display) | 4321 | all energy-rising steps occur here | **HIGH** |
| 5 | **Complete / target reached** (still plugged) | 255 | flat energy, appears at end of active charge with cse 3→5 | MED |
| 1 | **Idle / off / standby** | 323 | only in unplugged tails (co-occurs with plug=1) | MED |

### `evseType`
**No data.** Field absent from all 4899 frames (AC L2 home sessions carry no `evseType`). Meaning
undetermined — expect it only on DC fast / Rivian-network sessions. Confidence: N/A.

### Canonical per-session sequence observed
`(cse=3,plug=2,disp=3)` charging [energy rising] → `(3,2,5)` then `(5,2,5)` at target [energy flat, still
plugged] → **plug flips 2→1, disp→1** on unplug → frozen tail `(5,1,1)` or `(1,1,1)` [power/energy held].

### Recommended dashboard derivation (SPEC for follow-up code)
- **`isCharging = chargingStateEnum === 3`** (optionally corroborate with a recent rise in
  `totalChargedEnergyKwh`). **Do NOT use `powerKw > 0` alone** — power freezes non-zero for hours after a
  session ends and would report charging while the car is unplugged. Here the enum is strictly more
  reliable than power.
- **`isPluggedIn = plugConnectionStatus === 2`** (equivalently `!== 1`). Clean, reliable; `plug=1` only
  ever co-occurs with the idle tail.
- **`isChargeComplete = chargingStateEnum === 5`** (best-effort; distinguishes "at target, done" from the
  idle `cse=1`).
- Treat `powerKw` / `totalChargedEnergyKwh` as *display values only* while `chargingStateEnum !== 3`;
  they are stale in every non-charging state.
