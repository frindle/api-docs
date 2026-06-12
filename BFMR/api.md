# BFMR API

BFMR exposes two separate APIs with different base URLs and authentication:

| Surface | Base URL | Auth |
|---------|----------|------|
| REST API | `https://api.bfmr.com/api/v2` | API-KEY / API-SECRET headers |
| Web App API | `https://www.bfmr.com/api` | JWT Bearer token (from login) |

---

## REST API (`api.bfmr.com`)

### Authentication

All requests require:

| Header | Value |
|--------|-------|
| `API-KEY` | Your BFMR API key |
| `API-SECRET` | Your BFMR API secret |

Credentials stored as `bfmr_api_key` and `bfmr_api_secret` in user settings.

---

## Web App API (`www.bfmr.com`)

### Authentication

JWT Bearer token obtained from `POST /api/login`. Expires in ~30 days.

#### `POST /api/login`

```json
{ "email": "...", "password": "..." }
```

Response field: `access_token` (also check `token`). Use as `Authorization: Bearer <token>`.

POST requests also require the XSRF-TOKEN cookie value sent as `X-XSRF-TOKEN` header (Laravel CSRF protection). The cookie is set by the login response.

Credentials stored as `bfmr_email` and `bfmr_password` in user settings.

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

---

### Tracker (REST API)

#### `GET /my-tracker`

Get the user's current tracker items.

**Query params:** `page_size`, `page_no`, `quick_filter` (`all`|`pending`|`action_needed`|`paid`|`closed`), `status`, `search`, `start_date`, `end_date`, `order_by`, `order`

---

### Tracker (Web App API)

#### `GET /api/my-tracker`

Fetch tracker rows with richer fields than the REST endpoint. Used for tracking submission matching.

**Query params:** `page_size`, `page_no`, `start_date`, `end_date`, `filter_tab` (`all`|`action_needed`|...), `filter_status`

**Response:** `{ data: TrackerRow[] }` (also check `tracker`, `my_tracker`, `items` keys)

```ts
interface TrackerRow {
  id: number;              // same as PID
  PID: number;
  RID?: number;            // reservation ID
  SID?: number | null;     // shipment ID
  my_tracker_id: number;
  type: string;            // "purchased" etc.
  item_id: number;
  deal_id: number;
  order_id: string;        // retailer order number — match against Order.orderNumber
  tracking_number: string; // empty string if not yet set
  qty: string;
  status: string;          // "reserved" | "purchased" | "processed" | "paid" etc.
  reserved_at: string;     // "MM/DD/YYYY"
  retail_price: number;
  sub_total: number;
  amount_paid: string;
  paid_at: string;
  qty_received: string;
  scanned_at: string;
  notes: string;
  has_custom_columns: number;
  is_bundle: number;
  force_delete_shipment_after_deadline?: number;
}
```

---

#### `POST /api/my-tracker`

Submit or update tracker rows (used to set tracking numbers).

**Headers:** `Authorization: Bearer <token>`, `X-XSRF-TOKEN: <xsrf>`

**Request:**
```json
{
  "tracker_data": [
    {
      "id": 1420264,
      "PID": 1420264,
      "RID": 1625191,
      "SID": null,
      "type": "purchased",
      "force_delete_shipment_after_deadline": 0,
      "item_id": 12582,
      "qty": "3",
      "my_tracker_id": 4368259,
      "notes": "",
      "order_id": "ORDER-NUMBER-HERE",
      "tracking_number": "1Z123ABC1234567890",
      "deal_id": 9113,
      "has_custom_columns": 0,
      "is_bundle": 0,
      "amount_paid": "-",
      "paid_at": " - ",
      "qty_received": "-",
      "reserved_at": "06/11/2026",
      "retail_price": 293,
      "scanned_at": " - ",
      "status": "purchased",
      "sub_total": 879
    }
  ],
  "dateRange": { "start": "2026-03-11", "end": "2026-06-11" }
}
```

**Flow:** GET tracker rows → find row where `order_id` matches our order number → POST the full row back with `tracking_number` set.

**Note:** Send the complete row object as received from GET — do not strip fields. The `dateRange` is a ~3-month window matching the GET query.

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
| `lib/bfmr.ts` | REST API client — API-KEY/SECRET auth, deals, reservations, shipments |
| `lib/bfmrWeb.ts` | Web App API client — JWT login, profile fetch, tracker fetch, tracking submission |
| `app/api/bfmr/web-login/route.ts` | POST handler — logs in with email/password, fetches and saves API key/secret from profile |
