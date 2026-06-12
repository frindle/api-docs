# BigSkyBuyers API

BigSkyBuyers uses **tRPC** served from their Next.js app. All API calls go to `/api/trpc/{procedure}` with a `batch=1` query param. Authentication is cookie-based (session cookies from browser login).

**Base URL:** `https://www.bigskybuyers.com/api/trpc`

## Authentication

Cookie-based. The browser's session cookies are sent automatically via `credentials: "include"`. No explicit token management needed.

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
