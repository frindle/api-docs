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
    distanceToEmpty { timeStamp value }     # miles
    batteryLimit { timeStamp value }        # 0-100 percent
    timeToEndOfCharge { timeStamp value }   # minutes
    chargerState { timeStamp value }        # see values below
    chargerStatus { timeStamp value }
    powerState { timeStamp value }
    vehicleMileage { timeStamp value }      # miles
    doorFrontLeftLocked { timeStamp value } # "locked" | "unlocked"
    doorFrontRightLocked { timeStamp value }
    cabinPreconditioningStatus { timeStamp value } # "system_idle" | "system_on" | ...
    chargePortState { timeStamp value }
  }
}
```

**chargerState values:**
- `disconnected` — not plugged in
- `not_charging` — plugged in, not charging
- `charging_ac_1ph` — charging single-phase AC
- `charging_ac_3ph` — charging three-phase AC
- `charge_complete` — fully charged

**cabinPreconditioningStatus values:**
- `system_idle` — climate off
- `system_on` — climate running

**Note:** All values are strings except numeric ones (`batteryLevel`, `distanceToEmpty`, etc.).  
`distanceToEmpty` and `vehicleMileage` are in **miles** — multiply by 1.60934 for km.

---

## Vehicle Commands

Commands require HMAC signing with:
- Vehicle public key (`vas.vehiclePublicKey` from `currentUser`)
- Phone private key (enrolled phone)
- `vasPhoneId`, `deviceId`, `vehicleId`

This makes commands complex to implement without the BLE key pair. The HA integration (`bretterer/home-assistant-rivian`) has a working implementation for reference.

**For the dashboard, vehicle state is read-only until command signing is implemented.**

---

## Notes
- No public/official API — reverse engineered from iOS app (v707)
- `apollographql-client-name` header is required or requests are rejected
- Tokens appear session-based; no documented expiry time
- Source references: `bretterer/rivian-python-client`, `bretterer/home-assistant-rivian`
