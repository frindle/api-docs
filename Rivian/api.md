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
    distanceToEmpty { timeStamp value }     # miles (US vehicles)
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
- `distanceToEmpty` — miles (US vehicles)
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
