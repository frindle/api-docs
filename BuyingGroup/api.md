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

---

### Commitments

#### `POST /v1/commitment/get_commitments`

Lists active (and possibly historical) commitments for the authenticated buyer.

**Request:** `{}` (additional filters TBD — captured with empty body)

**Response payload structure** (one entry per commitment):

```json
{
  "status": "SUCCESS",
  "payload": {
    "commitments": [
      {
        "key": "uuid",
        "commitment_id": "CM-264497594",
        "user": { "key": "uuid", "user_id": "8066", "first_name": "...", "last_name": "...", "email": "..." },
        "deal": { "key": "uuid", "title": "Apple - Airpods Max 2 (Usb-C) - Blue", "deal_id": "DL-06260061" },
        "deal_id": "DL-06260061",
        "item": {
          "key": "uuid",
          "item_id": "Apple-Airpod-Max-2-2026-Usb-C-Blue-9305",
          "image": "<legacy>",
          "image_new": "https://storage.googleapis.com/bg-prod-storage/.../image.webp"
        },
        "deal_status": true,
        "tracking_linked_required": false,
        "tracking_link_expiry_date": null,
        "allow_overcommit_request": false,
        "deal_exclude_from_tier_calculation": false,
        "exclude_from_tier_calculation": false,
        "commitment_order_count": null,
        "status": "ACTIVE",
        "count": 3,
        "fulfilled": 0,
        "expiry_day": "07-11-2026",
        "price": "399.00",
        "commission": "0.00",
        "new_price": null,
        "new_commission": null,
        "new_bg_points_reward": null,
        "total": "1197.00",
        "bg_points_reward": 0,
        "editable": true,
        "created_dt": "06-22-2026, 08:50:27",
        "new_price_change_dt": null,
        "commitment_edit_count_request": null,
        "is_in_scanning_grace_period": false,
        "split_from": null,
        "split_reason": null
      }
    ]
  }
}
```

**Key fields for short-detection:**

| Field | Meaning |
|-------|---------|
| `count` | Total quantity committed to deliver |
| `fulfilled` | Quantity BG has recorded as delivered |
| `expiry_day` | `MM-DD-YYYY` deadline for delivery |
| `price` | Per-item price string (USD) |
| `total` | `price × count` string |
| `status` | `ACTIVE` while open; other values TBD |
| `tracking_linked_required` | If true, BG expects tracking link before counting fulfilled |

---

### Notifications

#### `GET /v1/notification/get_buyer_notification_count`

Lightweight polling endpoint for unread notification count.

**Response:** `{ "status": "SUCCESS", "payload": { "notification_count": 0 } }`

## Integration Files

| File | Purpose |
|------|---------|
| `lib/buyinggroup.ts` | API client — auth, orders, receipts, payments |
