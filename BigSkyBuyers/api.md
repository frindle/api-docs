# BigSkyBuyers API

BigSkyBuyers uses **tRPC** served from their Next.js app. All API calls go to `/api/trpc/{procedure}` with a `batch=1` query param. Authentication is cookie-based (session cookies from browser login).

**Base URL:** `https://www.bigskybuyers.com/api/trpc`

## Authentication

Cookie-based session, issued by **better-auth** with the **email-OTP** plugin.
Two calls, no password:

### 1. Send login code
```
POST /api/auth/email-otp/send-verification-otp
Content-Type: application/json

{ "email": "you@example.com", "type": "sign-in" }
```
→ `200` and emails a 6-digit code. (Observed 2026-07-06.)

### 2. Verify code → session cookie
```
POST /api/auth/sign-in/email-otp
Content-Type: application/json

{ "email": "you@example.com", "otp": "123456" }
```
→ `200` with `Set-Cookie` session token(s) (better-auth `*session_token*`).
Capture the `name=value` pairs and send them as `Cookie:` on subsequent
tRPC calls.

**Captcha caveat:** the public `/auth/signin` page also fires a Cloudflare
Turnstile check via `POST /api/auth/verify-captcha` (token format `1.xxx`,
returns `{"success":true}`). In captures the two auth POSTs above succeeded
**without** a captcha header, so a pure server-side flow works — but if
BigSky later enforces Turnstile on the auth API these will start 403ing and
a browser-captured cookie is the only fallback. The resell-tracker
`bigsky_cookie` setting accepts either source.

Analytics noise to ignore on the dashboard load: repeated
`POST /analytics/api/send` (umami) and duplicate `/_next/data/.../*.json`
prefetches — neither is part of the API surface.

## Endpoints

### `tracking.submitTracking`

Submit tracking numbers.

**Method:** POST  
**URL:** `https://www.bigskybuyers.com/api/trpc/tracking.submitTracking?batch=1`

**Request body:**
```json
{ "0": { "json": { "trackingNumbers": ["1Z...", "9..."] } } }
```

**Response:** tRPC batch response — check `[0].result.data.json` for result.

---

### `scan.getScanByUser`

Get all scans (submitted cards/items) for the current user.

**Method:** GET  
**URL:** `https://www.bigskybuyers.com/api/trpc/scan.getScanByUser?batch=1&input=<encoded>`

**Input parameter** (URL-encoded JSON):
```json
{ "0": { "json": null, "meta": { "values": ["undefined"] } } }
```

**Response:**
```json
[{ "result": { "data": { "json": [ ...ScanItem[] ] } } }]
```

**ScanItem fields:**

| Field | Type | Description |
|-------|------|-------------|
| `trackingNumber` | string | Carrier tracking number |
| `itemName` | string | Product/item name |
| `lineTotal` | number | Payment amount for this line |
| `scanDate` | string (ISO) | Date the item was scanned |
| `paymentDate` | string (ISO) \| null | Date payment was sent |

**Note:** Multiple scan items can share a `trackingNumber` (one per line item in the shipment). Sum `lineTotal` across items with the same tracking number to get the total sale price for that shipment.

## Integration Files

| File | Purpose |
|------|---------|
| `lib/bigsky.ts` | Tracking submission |
| `src/content/bigskybuyers.ts` | Browser extension content script — fetches scan history |
