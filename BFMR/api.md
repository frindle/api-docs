# BFMR API

Three separate surfaces with different base URLs and authentication:

| Surface | Base URL | Auth | Used by |
|---------|----------|------|---------|
| REST API | `https://api.bfmr.com/api/v2` | `X-API-KEY` + `X-API-SECRET` headers | Site (server-to-server) |
| Web App API | `https://www.bfmr.com/api` | JWT Bearer token (from login) | Site (browser) |
| TrackForMe Extension API | `https://hr-ext-api.bfmr.com/api` | `Bearer <extToken>` (UUID, not api_key) | Chrome extension |

---

## Site APIs

### Authentication

#### Web App API — `POST /api/login`

**Required body** — the `remember` field is mandatory; omitting it returns `422`:
```json
{ "email": "...", "password": "...", "remember": false }
```

**Response shape:**
```json
{ "success": true, "status": 200, "message": "...", "data": { "token": "<JWT>" } }
```

Extract token from `data.token` (nested under `data`, NOT a top-level `access_token`).

Use as `Authorization: Bearer <token>` on all subsequent requests.

POST requests also require the XSRF-TOKEN cookie value sent as `X-XSRF-TOKEN` header (Laravel CSRF protection). The cookie is set by the login response.

Credentials stored as `bfmr_email` and `bfmr_password` in user settings.

#### REST API — `X-API-KEY` + `X-API-SECRET`

All requests require:

| Header | Value |
|--------|-------|
| `API-KEY` | Your BFMR API key |
| `API-SECRET` | Your BFMR API secret |

Credentials stored as `bfmr_api_key` and `bfmr_api_secret` in user settings.

#### `GET /api/user/profile?_ts=<Date.now()>` (Web App API)

Fetch the authenticated user's profile, including API key and secret.

**Headers:** `Authorization: Bearer <token>`, `Cookie: <cookieStr>`

**Response shape:**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": 123,
      "email": "...",
      "api_access": {
        "api_key": "...",
        "api_secret": "..."
      }
    }
  }
}
```

Extract: `data.user.api_access.api_key` and `data.user.api_access.api_secret`.

The `_ts` query param is a timestamp — the server ignores its value; it just busts cache.

---

### Deals

#### `GET /api/deals?source=deals&tag=all&page=N&per_page=15&_ts=<Date.now()>` (Web App API)

Paginated deals list.

**Response:**
```json
{
  "success": true,
  "status": 200,
  "message": "Deals list.",
  "data": {
    "showPrivate": false,
    "deals": [ ...DealListItem[] ]
  }
}
```

**DealListItem fields (confirmed):**

| Field | Type | Notes |
|-------|------|-------|
| `id` | number | Deal ID |
| `title` | string | Deal title |
| `slug` | string | URL slug — used in detail/reserve endpoints |
| `code` | string | e.g. `"D-T7WMQ"` |
| `subtitle` | string \| null | Short instructions |
| `value` | string | Payout value, e.g. `"158.00"` |
| `retail_type` | string | `"Full Retail"` \| `"Below Retail"` \| `"Above Retail"` |
| `retail_price` | string \| null | Retail price if known |
| `cashback` | null | Not observed with a value |
| `above_retail_amount` | string \| null | Extra above retail |
| `back_in_sale` | 0\|1 | |
| `is_reservation_closed` | 0\|1 | Whether reservations are closed |
| `reservation_closed_at` | string \| null | |
| `other_retailers` | 0\|1 | Whether other retailers are allowed |
| `address_id` | number | Primary ship-to address |
| `available_addresses` | string | JSON-encoded array of address IDs |
| `status` | string | `"active"` etc. |

---

#### `GET /api/deals/{slug}/items-reservations?isTracker=0&_ts=<Date.now()>` (Web App API)

Deal detail with items and per-item reservation state. Called when opening a deal page.

**Response:**
```json
{
  "success": true,
  "data": {
    "deal": {
      "id": 9175,
      "title": "...",
      "multiple_items": 0,
      "deal_value": "158.00",
      "slug": "...",
      "picture_url": "https://d2d0gtcnbylgtv.cloudfront.net/...",
      "picture_url_mobile": "...",
      "picture_url_thumb": "...",
      "deadline": false,
      "items": [ ...DealItem[] ]
    }
  }
}
```

**DealItem fields (confirmed):**

| Field | Type | Notes |
|-------|------|-------|
| `item_id` | number | |
| `item_name` | string | Full item name |
| `color_name` | string | e.g. `"Black"` |
| `color_code` | string | hex e.g. `"#000000FF"` |
| `model_number` | string | |
| `band_size` | null | For ring/watch deals |
| `reserved_qty` | string | User's current reserved quantity (empty string = none) |
| `max_can_reserve` | number | How many more the user can reserve |
| `enable_request_more_cta` | boolean | Whether to show "request more" button |
| `remaining_reservations` | number | Total remaining slots globally |
| `reserve_max_avail_qty` | number | Max quantity per reservation request |
| `is_user_reserved` | number | Count of items the user has reserved (e.g. `3`); not a boolean |
| `tracking_deadline` | string \| null | |
| `has_48_hrs` | 0\|1 | |
| `deal_value` | string | Same as deal-level value |
| `links` | DealLink[] | Vendor links with stock status and purchase URL |

**DealLink fields (confirmed):**

| Field | Type | Notes |
|-------|------|-------|
| `id` | number | |
| `deal_id` | number | |
| `vendor_id` | number | |
| `item_id` | number | |
| `price` | string | |
| `status` | number | |
| `out_of_stock` | 0\|1 | |
| `in_stock` | boolean | |
| `vendor.name` | string | e.g. `"Walmart"` |
| `vendor.slug` | string | |
| `vendor.url` | string | |
| `vendor.logo_url` | string | |
| `item_link.link_url` | string | Purchase URL (may be redirect/affiliate link) |
| `item_link.identifier` | string | Retailer product identifier |

**Additional deal-level fields (confirmed):**

| Field | Type | Notes |
|-------|------|-------|
| `current_limit` | number | User's current global reservation limit (same as `max_can_reserve`) |
| `retailers` | `{name, slug, logo, vendor_order, in_stock}[]` | Active retailers for this deal |
| `item_colors` | `{name, code}[]` | Unique colors across items |
| `is_reservation_closed` | 0\|1 | |
| `reservation_closed_at` | string \| null | |
| `is_increase_request_pending` | boolean | |
| `any_tracking_deadline_set` | boolean | |
| `is_tracking_deadline_same` | boolean | |
| `tracking_deadline_all` | 0\|1 | |
| `value` | string | Same as `deal_value` |
| `retail_type` | string | `"Full Retail"` \| `"Below Retail"` \| `"Above Retail"` |
| `retail_price` | string \| null | |
| `above_retail_amount` | string \| null | |
| `back_in_sale` | 0\|1 | |
| `other_retailers` | 0\|1 | |
| `address_id` | number | |
| `available_addresses` | string | JSON-encoded array of address IDs |
| `code` | string | e.g. `"D-T7WMQ"` |
| `ends_at` | string \| null | |
| `new_till` | string | Date until deal is marked "new" |
| `subscribed` | boolean | Whether user is subscribed to deal alerts |
| `still_on_sale` | 0\|1 | |
| `closed` | boolean | |
| `combine_items` | 0\|1 | |

---

#### `POST /api/deals/reserve` (Web App API)

Reserve items on a deal.

> **Request body not yet captured from spy** — structure unknown. Known to include deal/item identifiers and quantity.

**Response (confirmed):**
```json
{
  "success": true,
  "status": 200,
  "message": "Quantity reserved successfully.",
  "data": {
    "invalid_items": [],
    "current_limit": 26,
    "enable_request_more_cta": true,
    "is_reservation_closed": false,
    "has_retailers": true,
    "deadline_modal": 1
  }
}
```

---

#### `GET /api/deals/{slug}/retailer-links-deal-detail?_ts=<Date.now()>` (Web App API)

Called immediately after a successful `POST /api/deals/reserve`. Returns current vendor stock status and updated per-item reservation state.

**Response:**
```json
{
  "success": true,
  "data": {
    "vendors": [
      {
        "vendor_id": 2,
        "vendor_name": "Walmart",
        "vendor_logo": "https://...",
        "vendor_order": 2,
        "in_stock": true
      }
    ],
    "items": [
      {
        "item_id": 15298,
        "reserved_qty": "",
        "enable_request_more_cta": true,
        "remaining_reservations": 23,
        "reserve_max_avail_qty": 1,
        "is_user_reserved": 3,
        "color_name": "Black",
        "color_code": "#000000FF",
        "model_number": "43Q31K",
        "band_size": null,
        "tracking_deadline": null,
        "is_expired": 0,
        "has_48_hrs": 1
      }
    ],
    "identifiers": [
      [
        {
          "item_name": "...",
          "identifier": "15822164529",
          "link_url": "https://ftc.cash/r5Rn8",
          "closing_status": false,
          "closing_date": null,
          "in_stock": true,
          "item_id": 15298,
          "vendor_id": 2
        }
      ]
    ]
  }
}
```

`is_user_reserved` reflects the updated count after reservation. `identifiers` is a 2D array (one sub-array per item) of retailer product links with in-stock status.

---

### Tracker

#### `GET /api/my-tracker` (Web App API)

Fetch tracker rows with richer fields than the REST endpoint. Used for tracking submission matching.

**Query params:** `page_size`, `page_no`, `start_date`, `end_date`, `filter_tab` (`all`|`action_needed`|...), `filter_status`

**Response:** `{ data: TrackerRow[] }` (also check `tracker`, `my_tracker`, `items` keys)

```ts
interface TrackerRow {
  id: number;              // = RID for reservations, = PID for purchased
  PID: number | null;      // purchase ID — null for reservations
  RID: number | null;      // reservation ID — null for purchased
  SID?: number | null;     // shipment ID
  my_tracker_id: number;
  type: string;            // "purchased" | "reservation"
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
  custom_columns: unknown[];
  is_bundle: number;
  force_delete_shipment_after_deadline?: number;
}
```

---

#### `POST /api/my-tracker` (Web App API)

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

**Reservation-type row example** (setting order number on a `status: "reserved"` row):
```json
{
  "tracker_data": [
    {
      "id": 1628292,
      "PID": null,
      "RID": 1628292,
      "SID": null,
      "type": "reservation",
      "force_delete_shipment_after_deadline": 0,
      "item_id": 15298,
      "qty": "3",
      "my_tracker_id": 4375151,
      "notes": "",
      "order_id": "ORDER-NUMBER-HERE",
      "tracking_number": "",
      "deal_id": 9175,
      "has_custom_columns": 0,
      "custom_columns": [],
      "is_bundle": 0,
      "amount_paid": "-",
      "paid_at": " - ",
      "qty_received": "-",
      "reserved_at": "06/13/2026",
      "retail_price": 158,
      "scanned_at": " - ",
      "status": "reserved",
      "sub_total": 474
    }
  ],
  "dateRange": { "start": "2026-03-13", "end": "2026-06-13" }
}
```

**Flow:** GET tracker rows → find row where `order_id` matches our order number → POST the full row back with `tracking_number` set.

---

#### `PUT /api/my-tracker/action` (Web App API)

Cancel a reservation (and likely other tracker actions).

**Headers:** `Authorization: Bearer <token>`, `X-XSRF-TOKEN: <xsrf>`

**Request:**
```json
{
  "action": "cancel",
  "tracker_data": [
    {
      "id": 1628298,
      "type": "reservation",
      "force_delete_shipment_after_deadline": 0,
      "item_id": 15298,
      "qty": "3",
      "order_id": "",
      "retail_price": 158,
      "sub_total": 474,
      "tracking_number": "",
      "qty_received": "-",
      "paid_at": " - ",
      "amount_paid": "-",
      "reserved_at": "06/13/2026",
      "scanned_at": " - ",
      "notes": "",
      "status": "reserved",
      "deal_id": 9175,
      "SID": null,
      "RID": 1628298,
      "PID": null,
      "is_bundle": 0,
      "my_tracker_id": 4375158,
      "has_custom_columns": 0,
      "custom_columns": []
    }
  ],
  "dateRange": { "start": "2026-03-13", "end": "2026-06-13" }
}
```

**Response:**
```json
{ "success": true, "status": 200, "message": "Data updated successfully!", "data": null }
```

Pass the full tracker row as received from `GET /api/my-tracker` — same shape as `POST /api/my-tracker` but wrapped with `"action": "cancel"` and using PUT method.

**Note:** Send the complete row object as received from GET — do not strip fields. The `dateRange` is a ~3-month window matching the GET query.

Key distinction between `type: "purchased"` and `type: "reservation"`:
- `"purchased"`: `PID` = row ID, `RID` = the linked reservation ID, `id` = PID
- `"reservation"`: `PID` = null, `RID` = row ID, `id` = RID

---

#### `GET /api/get-active-items?_ts=<Date.now()>` (Web App API)

Returns all items that are currently active (used to populate tracker dropdowns).

**Response:**
```json
{
  "success": true,
  "status": 200,
  "message": "Active Items",
  "data": {
    "active_items": [
      {
        "id": 15298,
        "item_name": "TCL - 43\" Q31K Series 1080P FHD QLED Smart TV with Google TV - Black - 43Q31K",
        "item_model_number": "43Q31K",
        "item_color": {
          "id": 9,
          "name": "Black",
          "color_code": { "id": 85, "color_id": 9, "color_code": "#000000FF" }
        },
        "msrp": null
      }
    ]
  }
}
```

---

#### `GET /api/my-tracker/get-total-counts` (Web App API)

Returns aggregate totals for the current tracker filter (same query params as `GET /api/my-tracker`).

**Query params:** `page_size`, `page_no`, `start_date`, `end_date`, `filter_tab`, `filter_status`

**Response:**
```json
{
  "success": true,
  "status": 200,
  "message": "Tracker.",
  "data": {
    "total_reserved_count": "5",
    "sub_total": "1620.00",
    "quantity_received": 1,
    "amount_paid": 0
  }
}
```

---

### REST API Endpoints (`api.bfmr.com`)

> **Note:** The REST API and Web App API appear to have overlapping functionality. Prefer the Web App API (confirmed from spy) over the REST API where both exist.

#### `GET /my-tracker`

Get the user's current tracker items.

**Query params:** `page_size`, `page_no`, `quick_filter` (`all`|`pending`|`action_needed`|`paid`|`closed`), `status`, `search`, `start_date`, `end_date`, `order_by`, `order`

#### `GET /deal/reservations/active`

List active deal reservations for the authenticated user.

> **Response shape not yet confirmed from spy capture.**

#### `GET /deals`

> **Not confirmed from spy — use Web App API `GET /api/deals` instead.**

---

## TrackForMe Extension API (`hr-ext-api.bfmr.com`)

Used by the BFMR TrackForMe Chrome extension (`ikhaebolenfgboimhmdminlbjfebmbpo`) to push Amazon order + tracking data directly to BFMR.

**Auth:** `Authorization: Bearer <extToken>` where extToken is the UUID from `GET /api/get-amazon-extensions-token` (NOT the `api_key`).

#### `GET /api/get-amazon-extensions-token` (Web App API)

Fetch the current extension token (UUID).

**Response shape:**
```json
{ "success": true, "data": { "token": "8b5ef217-xxxx-xxxx-xxxx-xxxxxxxxxxxx" } }
```

Extract: `data.token`.

> **Warning:** `POST /api/reset-amazon-extensions-token` resets this token — calling it invalidates the live TrackForMe extension. Always use GET, never POST.

Stored as `bfmr_ext_token` in user settings.

---

### `POST /api/auth`

Verify the extToken is valid.

### `POST /api/orders`

Submit order data (including tracking numbers) scraped from Amazon.

**Request:** `{ "order_data": [...] }` — exact shape of each `order_data` item not yet confirmed from spy capture. The extension scrapes Amazon order pages and sends the structured data here.

### `POST /api/fetch-order-responses`

Poll for BFMR's responses on previously submitted orders.

---

## Known Unknowns

| # | What | Why needed |
|---|------|------------|
| 1 | `POST /api/deals/reserve` request body | Build reservation feature in the app |
| 2 | `GET /deal/reservations/active` response shape | Show user's active reservations |
| 3 | Exact `order_data` payload shape for `POST /api/orders` on `hr-ext-api.bfmr.com` | Rewrite `submitTracking()` to use TrackForMe API instead of `POST /api/my-tracker` |

---

## Integration Files

| File | Purpose |
|------|---------|
| `lib/bfmr.ts` | REST API client — `X-API-KEY`/`X-API-SECRET` auth, deals, reservations, shipments |
| `lib/bfmrWeb.ts` | Web App API client — JWT login, profile + extToken fetch, tracker fetch, tracking submission |
| `app/api/bfmr/web-login/route.ts` | POST handler — logs in with email/password, fetches and saves all 5 settings |
| `app/api/bfmr/full-sync/route.ts` | POST handler — bulk BFMR order sync, called by "Resync Groups" |

**Settings keys:** `bfmr_email`, `bfmr_password`, `bfmr_api_key`, `bfmr_api_secret`, `bfmr_ext_token`
