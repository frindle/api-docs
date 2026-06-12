# BFMR API

**Base URL:** `https://api.bfmr.com/api/v2`

## Authentication

All requests require two headers:

| Header | Value |
|--------|-------|
| `API-KEY` | Your BFMR API key |
| `API-SECRET` | Your BFMR API secret |

Credentials are stored as `bfmr_api_key` and `bfmr_api_secret` in user settings.

## Endpoints

### Deals

#### `GET /deals`

List available deals.

**Query params:** `page`, `page_size`

**Response:** Array of deal objects with `slug`, `retailer`, `cashback`, etc.

---

#### `GET /deals/{slug}`

Get details for a specific deal by slug.

---

#### `GET /deal/reservations/active`

List active deal reservations for the authenticated user.

**Response:** Array of reservations with deal info, quantities, and status.

---

### Tracker

#### `GET /my-tracker`

Get the user's current tracker items (purchases in progress).

---

#### `POST /my-tracker`

Add or update a tracker item.

---

### Shipments

#### `GET /shipments/status`

Check shipment status by tracking number.

**Query params:**

| Param | Description |
|-------|-------------|
| `tracking_number` | Carrier tracking number |

---

### Insurance

#### `GET /insurance/shipments`

List shipments eligible for or covered by insurance.

---

#### `POST /insurance/file`

File an insurance claim for a shipment.

---

### Lookup

#### `GET /look-ups/retailers`

List all retailers supported by BFMR deals.

**Response:** Array of retailer names/objects.

## Integration Files

| File | Purpose |
|------|---------|
| `lib/bfmr.ts` | API client — auth, deals, reservations, shipments |
