# BuyingGroup API

**Base URL (data):** `https://api.prod.buyinggroup.com/v1`
**Base URL (auth):** `https://api.prod.buyinggroup.com/v2`

## Authentication

Django SimpleJWT. Login returns an `access` (short-lived) and `refresh` (long-lived) token, both nested under a `token` key.

All data endpoints use `Authorization: Bearer {access_token}` and `Content-Type: application/json`.

### `POST /v2/token/get`

Login.

**Request:**
```json
{ "username": "email@example.com", "password": "..." }
```

**Response:**
```json
{ "token": { "access": "...", "refresh": "..." } }
```

---

### `POST /v2/token/refresh`

Refresh an expired access token.

**Request:**
```json
{ "refresh": "..." }
```

**Response:**
```json
{ "access": "..." }
```

## Data Endpoints

All data endpoints are POST with a JSON body (even when just listing/getting). Most accept `page` and `page_size` for pagination.

### Payments

#### `POST /v1/payment/get_payments`

List payments.

**Request:** `{}`

**Response:** Array of payment objects.

**Payment statuses:** `"PAID"`, `"REQUESTED"`, `"SENT"`

---

### Receipts

#### `POST /v1/receipt/get_receipts`

List receipts (paginated).

**Request:**
```json
{ "page": 1, "page_size": 50 }
```

---

#### `POST /v1/receipt/get_details`

Get full details for a single receipt.

**Request:**
```json
{ "receipt_id": 12345 }
```

---

### Orders

#### `POST /v1/order/get_orders`

List orders (paginated).

**Request:**
```json
{ "page": 1, "page_size": 50 }
```

---

#### `POST /v1/order/add_trackings`

Submit tracking numbers for an order. Uses multipart `FormData` — the `tracking_list` field is a JSON-encoded array.

**FormData field:**

| Field | Type | Value |
|-------|------|-------|
| `tracking_list` | string (JSON) | `[{"tracking_number": "1Z...", "order_id": 123}]` |

---

### Deals

#### `GET /v1/deal/get_deals_new`

List deals.

**Query params:**

| Param | Description |
|-------|-------------|
| `page` | Page number |
| `page_size` | Items per page |
| `data_type` | Filter by type |
| `buyer_view` | Set `true` for buyer-facing view |

---

### Dashboard

#### `POST /v1/dashboard/get_statistics`

Get summary statistics for the dashboard.

**Request:** `{}`

## Integration Files

| File | Purpose |
|------|---------|
| `lib/buyinggroup.ts` | API client — auth, orders, receipts, payments |
