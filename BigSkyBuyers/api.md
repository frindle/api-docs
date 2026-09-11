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

**Session rotation:** better-auth rotates session tokens on every authenticated
response — each reply includes fresh `Set-Cookie` headers with extended expiry.
The browser handles this automatically, but server-side callers must capture and
persist the updated cookies after each request or the session dies within ~1 hour.
The two cookies are `__Secure-better-auth.session_token` (~12h expiry) and
`__Secure-better-auth.session_data` (~10d expiry). Both are required.

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

---

### `tracking.getNotCheckedInTracking`

Tracking numbers that have been submitted but not yet scanned/checked in.
Backs the "Tracking Not Checked In" table on the dashboard.

**Method:** GET
**URL:** `https://www.bigskybuyers.com/api/trpc/tracking.getNotCheckedInTracking?batch=1&input=<encoded>`

**Input parameter** (URL-encoded JSON): same empty-input shape as `scan.getScanByUser` above.

**Response:**
```json
[{ "result": { "data": { "json": [ ...NotCheckedInRow[] ] } } }]
```

**NotCheckedInRow fields** (confirmed 2026-07-13 via a real submit → capture → delete cycle):

| Field | Type | Description |
|-------|------|-------------|
| `tracking` | string | The tracking number — **note the field is `tracking`, not `trackingNumber`** despite every other endpoint on this API using `trackingNumber`. Don't assume naming consistency across BigSky's procedures. |
| `trackingCreated` | string (ISO) | When the tracking number was submitted |
| `trackingId` | number | Internal row ID — needed for `tracking.deleteTracking` |
| `carrier` | string | Carrier name, or `"Unknown"` if undetected (observed on a manually-typed non-carrier test string) |

Row disappears from this list once BigSky scans it (at which point it appears in `scan.getScanByUser` instead).

---

### `tracking.deleteTracking`

Remove a submitted tracking number (e.g. entered in error).

**Method:** POST
**URL:** `https://www.bigskybuyers.com/api/trpc/tracking.deleteTracking?batch=1`

**Request body:**
```json
{ "0": { "json": { "trackingId": 13158, "trackingNumber": "TESTING123" } } }
```
Both `trackingId` (from `tracking.getNotCheckedInTracking`) and `trackingNumber` are required.

**Response:** confirms the deleted row, with a couple of fields not present in the list endpoint:
```json
{
  "trackingId": 13158, "tracking": "TESTING123", "tracking12": "TESTING123",
  "trackingPurchaser": "n8kmzi1jx9urhg1mzysec3ra", "trackingCreated": "2026-07-13T15:03:41.000Z",
  "trackingUpdated": null, "carrier": "Unknown"
}
```
`tracking12` and `trackingPurchaser`/`trackingUpdated` purpose unconfirmed beyond their names — not currently used by resell-tracker.

---

### Other observed endpoints (unintegrated)

Seen in the same batched dashboard-load call (`payment.getUserPayments,tracking.getNotCheckedInTracking,payment.getUserBalances,scan.getScanByUser,scan.getPaidThroughDate?batch=1&input=...`), not yet wired into resell-tracker:

- **`payment.getUserBalances`** → `[{ "balance": "4795.41" }]` — current account balance.
- **`payment.getUserPayments`** → array of `{ paymentAmount, paymentCreated, paymentId, paymentPurchaser, lastPaymentCreated, paymentUpload }` — payment batch history.
- **`scan.getPaidThroughDate`** → `[{ "paidThroughDate": "2026-06-22T02:31:07.000Z" }]` — the cutoff date payments have been settled through.

## Integration Files

| File | Purpose |
|------|---------|
| `lib/bigsky.ts` | Auth, tracking submission, server-side data fetching (fetchScanItems, fetchNotCheckedInTracking), session cookie refresh |
| `lib/bigskyHealth.ts` | Periodic session liveness check — validates cookie, warns on approaching expiry, flags dead sessions |
| `app/api/bigsky/sync-orders/route.ts` | Sync endpoint — accepts extension-pushed groups or fetches server-side (fetch:true). Per-item value splitting when one tracking spans multiple orders |
| `lib/autoSync.ts` | Scheduled sync loop — calls sync-orders with fetch:true every 6h for users with bigsky_cookie |
| `src/content/bigskybuyers.ts` | Browser extension content script — fetches scan history and not-checked-in tracking, pushes to `/api/bigsky/sync-orders` |
