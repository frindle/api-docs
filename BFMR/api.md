# BFMR API

> **⚠ AWS WAF block on `www.bfmr.com/api/*` — corrected 2026-10-06.** The 2026-09-23 version of this banner said the block is AWS WAF Bot Control bound to a browser-minted `aws-waf-token`, so writes "must be driven by a real browser". That attribution is **not supported by the newer measurement** and is retired. What is actually proven:
>
> **Proven (measured 2026-10-06, unauthenticated `GET https://www.bfmr.com/api/my-tracker` from the Mac, Node 26.7):**
>
> | Client | Result | Meaning |
> |--------|--------|---------|
> | Node default `fetch` / `node:https` | **403** `text/html`, `server: awselb/2.0` | blocked at the AWS ALB WAF, never reaches the app |
> | `node:https` with `{ ciphers: 'DEFAULT' }` | **401** JSON from nginx | passes the WAF, reaches the app (401 = unauthenticated, as expected) |
> | `node:https` with `{ maxVersion: 'TLSv1.2' }` | **401** JSON from nginx | same |
> | `curl` | **401** JSON | same |
>
> So the 403 is a rule on the **Node default TLS fingerprint** (cipher/handshake shape), not a missing `aws-waf-token`. The same request with a different cipher list flips 403 to 401 with no token involved. (The 2026-09-23 "401 despite correct headers" observation is an app-level response, i.e. the request had already passed the WAF; it is not evidence of token binding. The `awswaf_*` localStorage keys and the `token.awswaf.com` challenge/`mp_verify` calls are real and were observed on the page, but their role in blocking API calls is unproven.)
>
> **Still unproven / in progress:**
> * An **authenticated 200** through the `ciphers: 'DEFAULT'` path has not been confirmed live. Do not assume writes (`POST /api/my-tracker`, `POST /api/login`, `PUT /api/my-tracker/action`) work from server-side Node after the cipher change until that is measured.
> * The "login 403 = reCAPTCHA v3" claim (code comments in `lib/bfmrWeb.ts`, `sidecar/src/bfmr.js`) predates this finding and was **never re-measured**; it may be the same TLS block, may be reCAPTCHA, or both.
> * The server-side fix is **unfinished**: resell-tracker bundle `rt-bfmr-tls-fingerprint` is meant to make `loggedFetch` (`lib/apiCallLog.ts`) route `bfmr.com` hosts (only) through `node:https` with `ciphers: 'DEFAULT'`. As of 2026-10-06 nothing has landed on resell-tracker `main` (`lib/apiCallLog.ts` and `lib/bfmrWeb.ts` contain no cipher/`node:https` routing). Authoring jobs failed repeatedly (see "resell-tracker integration notes" below); the bundle was released from the queue lanes and is to be requeued. Live check after landing is via the app's own `/api/bfmr/deals`.
>
> **What works today:** the REST surface (`api.bfmr.com/api/v2`, `X-API-KEY`/`X-API-SECRET`) works from server-side Node (it is what the production sync uses). The headless **sidecar** works because it runs real Chromium and does the tracker fetch as an in-page `fetch()` (`page.evaluate`) inside an already-logged-in session — a browser TLS fingerprint, which is consistent with the finding. Manual workaround for pushing an order number the server cannot push: enter it on BFMR My Tracker.

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

> Shape caveat (2026-10-06): `app/api/bfmr/web-session-seed/route.ts` also accepts the raw login response as `data.user.auth_token`, so a newer login response may carry the token there instead of `data.token`. Unverified which is current; `lib/bfmrWeb.ts` `login()` reads `access_token`/`token` at top level or under `data`.

Use as `Authorization: Bearer <token>` on all subsequent requests.

POST requests also require the XSRF-TOKEN cookie value sent as `X-XSRF-TOKEN` header (Laravel CSRF protection). The cookie is set by the login response.

Credentials stored as `bfmr_email` and `bfmr_password` in user settings.

**Login gating + session handling (resell-tracker, verified in `lib/bfmrWeb.ts` 2026-10-06):** `POST /api/login` is described in code as reCAPTCHA v3 gated (server-side login returned 403; see the banner caveat that this was never re-measured against the TLS finding), so the normal path is a **manually seeded session**: a token captured from a real browser login is stored by `seedBfmrWebSession` (route `POST /api/bfmr/web-session-seed`) into settings `bfmr_session_token`, `bfmr_session_xsrf`, `bfmr_session_cookies`, `bfmr_session_expires`. BFMR now issues **~30-day JWTs** (captured live); `getSession` prefers the JWT's own `exp` claim (minus a 5-minute margin) over the stored expiry (commit f35fd16, 2026-09-20), falling back to a 50-minute cache TTL only for undecodable tokens. The sidecar path logs in through the shared manual-VNC flow instead (`sidecar/src/loginFlow.js`).

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

**Content-Type:** `application/x-www-form-urlencoded` (not JSON — spy tool cannot capture this)

**Request body:**
```
deal_slug=<redacted>
reservations[0][item_id]=15298
reservations[0][item_qty]=3
```

Multiple items use `reservations[0]`, `reservations[1]`, etc.

**Headers:** `Authorization: Bearer <token>`, `X-XSRF-TOKEN: <xsrf>`

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
  status: string;          // ⚠ LAGS — see status note below
  date_processed?: string; // set when the item is processed (authoritative)
  date_paid?: string;      // set when paid (authoritative)
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

#### ⚠ Status lifecycle — the `status` string lags; derive from fields

Package lifecycle (least → most advanced):
`(reserved/purchased) → shipped → pkg received → processed → paid`

**The `status` string does not reliably reflect the current stage** — an
item can show `shipped` on the tracker while BFMR's site already counts it
as processed, or `purchased` when it's really shipped. The authoritative
signals are the field values, so derive status from them and only fall back
to the string when none has fired:

| Real status | Signal |
|-------------|--------|
| paid | `date_paid` present |
| processed | `date_processed` present |
| pkg received | `qty_received > 0` |
| shipped | `tracking_number` present |
| (earlier) | fall back to `status` string |

**Terminal statuses win over the fields.** `cancelled`, `returned`, `return`, `set_aside`, `closed` are taken from the status string as-is (`BFMR_TERMINAL_STATUSES` in `lib/bfmr.ts`): a returned/cancelled row can still carry `date_processed`/`date_paid` from before the return, and deriving "processed" from those would resurrect it. For non-terminal rows the more advanced of (field-derived, string) wins, because the string is occasionally *ahead* (e.g. `paid` before `date_paid` is written). Split-shipment paid rollup (`computeBfmrPaidRollup`): an order's legs pay independently, so only the paid legs count toward paid money fields (order 900 regression: one paid $1893 leg + one shipped $631 leg read as fully paid).

An **order** spanning multiple items is only truly "processed" when *every*
active item is processed — roll up on the least-advanced item, not the most,
or the order reads Processed while items are still shipping. resell-tracker
implements this as `deriveBfmrStatus()` in `lib/bfmr.ts`.

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

**Split-shipment submission** (qty > 1 with multiple tracking numbers): send N entries in `tracker_data`, all sharing the same `id`/`PID`/`my_tracker_id`/`deal_id`/`item_id`/`order_id`, each with `qty: 1`, a distinct `rowIndex: 0..N-1`, and its own `tracking_number`. The minimal field set works (no need to echo every column from the GET row).

```json
{
  "tracker_data": [
    {
      "qty": 1,
      "id": 1485466,
      "my_tracker_id": 4570045,
      "deal_id": 9375,
      "item_id": 11175,
      "type": "purchased",
      "status": "purchased",
      "rowIndex": 0,
      "order_id": "114-6759124-7158630",
      "tracking_number": "9339589725265623497482"
    },
    {
      "qty": 1,
      "id": 1485466,
      "my_tracker_id": 4570045,
      "deal_id": 9375,
      "item_id": 11175,
      "type": "purchased",
      "status": "purchased",
      "rowIndex": 1,
      "order_id": "114-6759124-7158630",
      "tracking_number": "9339589725265624883161"
    }
  ],
  "dateRange": { "start": "2026-03-28", "end": "2026-06-28" }
}
```

Response: `{ success: true, data: { invalid_reservations: [], deal_suggestions: [], delete_shipment_deadline_warning: false } }`. After success, re-GET shows N separate rows with distinct `SID`s and `status: "shipped"`.

Lookup-before-submit pattern observed in the wild: `GET /api/my-tracker?...&filter_tab=all&search=<last4OfOrderNumber>` returns the matching reservation row; pull `id`/`my_tracker_id`/`deal_id`/`item_id` from there.

---

#### `PUT /api/my-tracker/action` (Web App API)

Cancel a reservation (and likely other tracker actions).

**Headers:** `Authorization: Bearer <token>`, `X-XSRF-TOKEN: <xsrf>`

**Request:**
```json
{
  "action": "cancel",
  "complete_package": false,
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

Top-level `complete_package` (confirmed 2026-09-23, present on a real cancel of a `type: "purchased"` row): meaning unconfirmed, observed `false` on a non-bundle single-item cancel (`is_bundle: 0`); worth re-checking on a bundled/multi-item reservation.

**Response:**
```json
{ "success": true, "status": 200, "message": "Data updated successfully!", "data": null }
```

Pass the full tracker row as received from `GET /api/my-tracker` — same shape as `POST /api/my-tracker` but wrapped with `"action": "cancel"` and using PUT method.

**Note:** Send the complete row object as received from GET — do not strip fields. The `dateRange` is a ~3-month window matching the GET query.

Key distinction between `type: "purchased"` and `type: "reservation"`:
- `"purchased"`: `PID` = row ID, `RID` = the linked reservation ID, `id` = PID
- `"reservation"`: `PID` = null, `RID` = row ID, `id` = RID

**Reservation-split ID mechanics (confirmed 2026-09-23) — distinct from shipment-split above:**

When a multi-qty **reservation** gets an order number attached to only *part* of its quantity (a different split axis than the shipment-split covered above, which splits an already-purchased row by tracking number): `RID` is the **parent batch ID**, assigned once at the original `POST /api/deals/reserve` call and stable across every later mutation of that original reservation. `my_tracker_id` and `PID` are **per-split-portion IDs**, newly minted the moment a portion of the reservation gets its own order#/tracking. `id` mirrors whichever ID is current for that row — `RID` before a split assigns a `PID`, `PID` after. `SID` stays `null` until a shipment stage is reached. Confirmed live: two reservations of the same item/deal (`item_id`/`deal_id` identical, created via two separate `POST /api/deals/reserve` calls) show up as **two separate tracker rows**, not one split line — BFMR's UI does not visually distinguish "two independent reservations of the same item" from "one reservation split into portions"; only `RID` tells them apart (different `RID`s = independent reservations; same `RID`, different `PID`/`my_tracker_id` = one reservation split).

**Field-clearing convention:** to clear `order_id`/`tracking_number` on a `POST /api/my-tracker` update, send empty string `""` — not `null`, not field omission.

**Business rule:** order numbers must be uploaded within **8 hours** of reservation creation to lock in the reservation (UI shows a live countdown badge, e.g. "7h 42m").

**Replacing tracking on an already-shipped row (confirmed 2026-10-02, order 943, done by hand in the BFMR UI):** the UI does `POST https://www.bfmr.com/api/my-tracker` with the shipped row (`type: "shipment"`, `status: "shipped"`, with its `SID`, `RID`, `PID`, `my_tracker_id`) echoed back in `tracker_data` with the new `tracking_number`, plus `dateRange`. Response `{ "success": true, "message": "Data updated successfully!" }`. Clearing uses `tracking_number: ""`. No separate edit/delete-shipment endpoint was found; the only other write is `PUT /api/my-tracker/action {action:"cancel"}` (whole reservation). **BFMR may respond by replacing the whole reservation:** in order 943 the old reservation (`my_tracker_id` 5102424) was replaced by a new reserve id / `my_tracker_id` 5123566 (processed, new tracking), and the old local rows lingered as stale cards because sync never deletes vanished rows. The payload builder/picker exist (`lib/bfmrReplaceTracking.ts`: `buildReplaceTrackingPayload` refuses empty, unchanged, non-`shipment` or non-`shipped` rows; `pickShipmentRow`, commit e692a48) but **no route or UI calls them yet**.

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

> **Docs vs. reality, confirmed 2026-08-23:** BFMR's official published schema (below, from their own
> API reference with all responses expanded) for `GET /api/v2/my-tracker` does **not** list `my_tracker_id`
> or `order_id` as response fields. Production code (`lib/bfmr.ts`'s `getMyTrackerAll` → `sync-reservations/route.ts`
> in resell-tracker) has been reading `item.my_tracker_id` and `item.order_id` directly off real responses from
> this exact endpoint since 2026-07-31 and it works (`BfmrReservation.myTrackerId`/`bfmrOrderId` populate
> correctly, confirmed live repeatedly). **Conclusion: BFMR's official docs are incomplete for this endpoint even
> in their fully-expanded form** — not just collapsed-placeholder-blocked as found earlier. Trust the code's
> field usage (backed by live-synced `raw` JSON) over this schema when they disagree. This also revises the
> earlier hypothesis that `my_tracker_id`/`order_id` are Web-App-surface-only concepts — they are real REST
> fields BFMR simply never documented.

#### `GET /api/v2/my-tracker`

Get the user's current tracker items.

**Query params:** `page_size` (1-1000), `page_no`, `order_by` (`status`|`qty`|`retail_price`|`qty_received`|`amount_paid`|`reserved_at`|`sub_total`|`date_processed`|`date_paid`), `order` (`asc`|`desc`), `search`, `start_date`, `end_date`, `quick_filter` (`closed`|`all`|`action_needed`|`pending`|`paid` — ignored if `status` is also set), `insurance_status` (`not_eligible`|`not_insured`|`insured`), `status` (comma-separated: `purchased`|`reserved`|`return`|`payment_error`|`shipped`|`processed`|`set_aside`|`paid`|`cancelled`|`returned`|`closed`|`deadline`|`pkg_received`)

**Response shape (official docs, `my_tracker` array items) — see caveat above, incomplete:**
```json
{
  "message": "Tracker.",
  "my_tracker": [
    {
      "reserve_id": "string", "purchase_id": "string", "shipment_id": "string",
      "qty": 0, "amount_paid": 0, "deal_id": 0, "deal_title": "string",
      "item_id": "string", "item_model_number": "string",
      "order_date": 0, "order_deadline": 0, "qty_received": "string",
      "reserved_at": 0, "date_processed": 0, "date_paid": 0,
      "retail_price": 0, "payout_price": 0, "status": 0, "total_payout": 0,
      "tracking_deadline": "string", "deal_type": "string", "tracking_status": "string",
      "missed_deadline": true, "insurance_status": "string", "note": "string",
      "retailer_links": [{ "retailer": "string", "identifier_id": "string", "url": "string" }],
      "issue_with_item": [{
        "accepted_items": [{ "name": "string", "status": "string" }],
        "rejected_items": [{ "name": "string", "status": "string", "issue_with_reason": ["string"] }]
      }]
    }
  ]
}
```
Real responses also include `my_tracker_id` (number) and `order_id` (string) per confirmed production usage — absent above.

#### `POST /api/v2/my-tracker` — Make Updates in My Tracker

Update reservations/purchases/shipments (max 500 objects/request). Different required-field sets per action:
- **Update reservation**: `reserve_id`, `qty` (qty can only decrease); `purchase_id`/`shipment_id` null
- **Create purchase**: `reserve_id`, `order_no`, `qty`; `purchase_id`/`shipment_id`/`tracking_number` null
- **Update purchase**: `reserve_id`, `purchase_id`, `order_no`, `qty`; `shipment_id` null
- **Create shipment**: `reserve_id`, `purchase_id`, `order_no`, `tracking_number`, `qty`; `shipment_id` null
- **Update shipment**: `reserve_id`, `purchase_id`, `shipment_id`, `tracking_number`, `qty`

Body: `{ "tracker_data": [{ "reserve_id", "purchase_id", "shipment_id", "qty", "order_no", "tracking_number" }] }`
Response: `{ "message": "...", "reservations_response": { "valid_items": [...], "invalid_items": [...] } }`

#### `GET /api/v2/deal/reservations/active`

List active deal reservations for the authenticated user. Resolves earlier "not yet confirmed" status.

```json
{
  "message": "Active Reservation List.",
  "reservation_list": [
    {
      "item": [{ "item_id": "string", "item_name": "string", "model_number": "string", "upcs": {},
                 "color_name": "string", "retailer_links": [{ "retailer": "string", "identifier_id": "string", "url": "string" }] }],
      "deal_id": "string", "reserve_id": "string", "reserved_quantity": 10,
      "remaining_quota": 10, "reserved_at": "string", "expiry_after": "string"
    }
  ]
}
```

#### `GET /api/v2/deals` — Active Deals List

Query params: `page_size` (1-50), `page_no`, `retailer` (comma-separated slugs), `retail_type` (`above`|`full`|`below`), `exclusive_deals_only` (`0`|`1`), `in_stock` (`0`|`1`). Response: `{ "message", "deals": { deal_id, deal_code, retail_price, title, slug, closing_at, is_reservation_closed, retailers, retail_type, payout_price, deal_information, is_bundle, is_exclusive_deal, items: [{ id, name, color, model, upc, category, retailer_links }] } }`

#### `GET /api/v2/deals/{slug}` — Deal Details

Same shape as above under `"deal"` instead of `"deals"`; items also carry `tracking_deadline` and `upc` as an array of strings (not a single string).

#### `GET /api/v2/look-ups/retailers` — Active Retailers List

Query params: `page_size` (1-1000), `page_no`. Response: `{ "message", "retailers": { "name", "slug", "link" } }`

#### `POST /api/v2/my-tracker/purchase/cancel` — Cancel Purchased

Body (form-encoded): `purchase_id[]`. Response: `{ "message", "invalid_purchases": [...] }`

#### `POST /api/v2/my-tracker/reservation/cancel` — Cancel Reserved

Body (form-encoded): `reserve_id[]`. Response: `{ "message", "invalid_reservations": [...] }`

#### `GET /api/v2/shipments/status` — Shipment Status of Tracking Number

Query param: `tracking_number`. Response: `{ "message", "tracker_data": [{ "tracking_number", "status", "accepted_items": [{name,status}], "rejected_items": [{name,status,issue_with_reason:[string]}] }] }`. Rejected items include the rejection reason.

#### `POST /api/v2/shipments/otp/amazon` — Submit Amazon OTP

For pending packages. Body: `{ "otp_data": [{ "tracking_number", "delivery_date", "otp" }] }`

#### `POST /api/v2/shipments/do-not-process` — Do Not Process Shipment

Body (form-encoded): `tracking_number` (required), `item_name`, `qty`, `notes`. For items with no active deal, or that should be set aside instead of processed.

#### Insurance (`api.bfmr.com/api/v2/insurance/*`)

- `POST /insurance/file` — file insurance for a shipment. Body (form-encoded): `tracking_number` (required), `package_value` (optional, else calculated from shipment items), `first_name`/`last_name`/`address[address_1]`/`address[address_2]`/`address[city]`/`address[state]`/`address[country]`/`address[zip]` — **all independently optional**, each falls back to the value already on the account profile if omitted, and fails naming the specific missing field if neither is set. `state` = ISO 3166-2 subdivision code without country prefix (e.g. `NH`). `country` = ISO 3166-1 **alpha-3** (e.g. `USA`, not `US`). ⚠️ **Submitting name/address fields here updates the account profile itself, not just the insurance filing.**
- `GET /insurance/shipments/{tracking_number}` — insurance + item details for one shipment: `{ insurance: { shipment: { tracking_number, package_cost, insured_amount, cert_no, cert_link, claim_no, policy_no, status, items: [{item_id,name,qty,price}] } } }`
- `POST /insurance/void` — body: `tracking_number`. Response: `{ "message": "Void request submitted successfully..." }`
- `GET /insurance/shipments` — paginated list. Query: `page`, `per_page`, `sort_by`, `sort_order`, `tracking_numbers` (comma-separated filter). Response includes `insurance.shipments[]` and a `paging` object (`current_page`, `from`, `last_page`, `to`, `total`, `perPage`, `pages`, `first`, `last`, `prev`, `next`).

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

## resell-tracker integration notes (verified against code and git history, 2026-10-06)

### Sync policy (Penn's decisions 2026-10-02; `lib/bfmrSyncTrigger.ts`, `lib/bfmrSyncScope.ts`)

BFMR syncs are **manual only** — `lib/autoSync.ts` deliberately does not call BFMR (avoids bot-looking background traffic; CardCenter and BigSky still sync every 6h). Triggers pick a scope for `POST /api/bfmr/sync-reservations`; an unknown trigger falls back to `all`:

| Trigger | Scope | Pulls |
|---------|-------|-------|
| `order-open` (opening an unlinked order) | `pending` | narrow, cheap |
| `manual` (Sync / Resync Groups) | `open` | the 7 open statuses: `purchased, reserved, payment_error, shipped, processed, deadline, pkg_received` |
| `sidecar` (SYNC_BFMR, below) | `open` | same, plus the scraped `webRows` |
| `scheduled` | `all` | all 13 statuses: the open 7 plus `return, set_aside, paid, cancelled, returned, closed` |
| `force-web-backfill` | (escape hatch) | forces the myTrackerId web backfill for every still-null row |

* `getMyTrackerAll` runs the filters **sequentially**, and retries **HTTP 429** with `Retry-After` (capped at 30 s) or exponential backoff (1.5 s base, capped 20 s); HTML error bodies are trimmed to a short `BFMR 429: ...` message (commit 9c15fe9, `lib/bfmr.ts`, `lib/bfmrRetry.test.ts`). Firing every filter at once is what tripped the rate limit.
* **Vanished lookup** (`lib/bfmrVanished.ts`): a local *open* reservation missing from the open-status pull has probably moved to a finished status, so it is looked up by retailer order number via `GET /api/v2/my-tracker?search=<order_id>` (cap 5, sequential, hits filtered to exactly that `order_id` so an ignored `search` cannot inject rows). Unverified: that BFMR `search` really matches order numbers (failure mode = no hit, row stays stale until a full pull). The sync response reports `scope`, `vanishedLookups`, `vanishedHits`.
* `supersededRowIds` (commit 0c858aa) is a **pure function** marking local rows replaced by a new BFMR reservation (same `item_id`, different `reserve_id` head, not a split sibling, not a finished status, empty lookups never supersede). It is **not yet wired** into `sync-reservations` or the UI.

### Reservation identity

* A BFMR reservation **split** returns several lines that share one `reserve_id`. The local key is `lineKey = reserve_id|purchase_id|shipment_id` (head falls back to `purchase_id` then `shipment_id`); keying on `reserve_id` alone dropped split lines (`lib/bfmrReservationLineKey.ts`, commits f5f5e84/e659e69). Status progression (reserved → purchased → paid) mints new line keys, so one `reserve_id` can legitimately have several local rows at different statuses (e.g. 323938 / 327007 / 332383 seen live 2026-10-06; whether any are true duplicates is an open question, no diagnosis queued yet).
* **myTrackerId backfill.** The REST surface returns `my_tracker_id` for most rows, but some (notably "Purchased / Enter tracking") never get one from REST. They are backfilled from the Web App tracker rows by a join on `reserved_at + item model/name + qty + order_id` (`lib/bfmrJoin.ts`; there is no shared id between the two surfaces), **only when exactly one Web row matches** (ambiguous/unmatched stay null, "loudly wrong, not silently wrong"; `matchSplitGroups` handles split portions). Eligibility: rows reserved on/after `2026-09-01` are retried every sync; older null rows are permanently excluded (333 of 759 measured 2026-09-09 as unmatchable, mostly outside BFMR's tracker history window); unparseable dates fall back to a 24 h retry window. The Web rows come from the sidecar scrape (`webRows` in the POST body) or, absent that, a server-side login (`getWebTrackerRows`).
* Link guards (`lib/bfmrLinkGuard.ts`, `bfmrLinkReconcile.ts`, `bfmrCrossOrderLinks.ts`): a reservation may not carry two links with the same tracking number (scoped **per reservation**, since several reservations of one order can legitimately ship under one tracking number); summed link quantity may not exceed the reservation qty; a link whose reservation's own `order_id` contradicts the order is refused/excluded from price (digits-only, bidirectional, 7-digit floor; unknown on either side is never a contradiction). `selectCanonicalBfmrLinks` drops no-tracking "parent" links once a reservation has tracked split children and resolves duplicate trackings by reservation endorsement, not by link id (the old order-wide rule halved order 929). Stale-link reconciliation moves links from a bare (no purchaseId/shipmentId) row to its single qty-matching live sibling; the sync response reports `staleLinkMigrations` and `staleLinkValues`.

### Order-number push and tracking submit

* `lib/bfmrPushGate.ts`: whether linking an order should push its order number to BFMR is decided on the **reservation's recorded order number** (not on "first time this link row is created", which silently skipped every edited link). It needs `myTrackerId`. BFMR has no split endpoint: posting a smaller `qty` against the row **is** the split, so a partial link yields a partial push.
* Tracking submit (`POST /api/bfmr/submit-reservation-tracking`): the POST is accepted-then-recorded locally immediately; the read-back verify is bounded (8 s fetch timeout, 30 s submit timeout) and non-blocking (`lib/bfmrSubmitFlow.ts`, commit 1a82bd4). Submit is not offered when the reservation has no tracker id (b68224c); BFMR tracking is never auto-submitted (3b00be6).
* **Server-side pushes from the app currently fail with `BFMR fetch tracker 403`** for the reason in the banner (Node TLS fingerprint on the web surface). A planned sidecar `BFMR_PUSH_ORDER_NUMBER` command (plan `rt-bfmr-push-via-sidecar`) is **not** in `EXTENSION_COMMAND_TYPES` on `main`; the allow-list there is `SYNC_AMAZON, SYNC_WALMART, SYNC_COSTCO, SYNC_BFMR, SYNC_BIGSKY, SCRAPE_CBM, SYNC_AMAZON_ORDER`. "reservation has no myTrackerId" means run the sidecar sync (main-page "Resync Groups"), not the order-page button.

### Sidecar SYNC_BFMR (`sidecar/src/bfmr.js`)

* Login is the shared manual-VNC flow; `installInterceptor` passively records the page's own `Authorization` / `X-XSRF-Token` headers on `bfmr.com/api/my-tracker` calls, and `confirmLoggedIn` requires a captured 200 with the real `data.my_tracker` array shape. Replaying the page's own headers fixed `fetch tracker 401` (commit 8ae1511).
* `syncBfmr` loads `https://www.bfmr.com/my-tracker`, then pulls the grid with an in-page `fetch` (3-month window, `page_size` 500, up to 10 pages, `filter_tab=all`, a status list that also includes `In Review` and `Not Received`) and POSTs the rows to `/api/bfmr/sync-reservations` as `webRows` (trigger `sidecar`); it **fails loudly** when the tracker accepts none (`6f81177`, `7e8deed`). Queued via ExtensionCommand `SYNC_BFMR`; "Resync Groups" absorbed the old Sync BFMR button (`0e5f8c5`).
* Stale in-file comment: the inline note at ~line 85 still says the page's cookies include `aws-waf-token`; the function header (commit 2f3293c, 2026-10-06) is the corrected one.

### Not landed yet (so do not document as behaviour)

* **Order-unlinked flag** (`bfmr-order-unlinked-flag`; `lib/trackingUnlinked.ts` is not on `main`; deferred behind `rt-order-mismatch-port` because both touch `app/orders/page.tsx`).
* **Link-sync feedback** (`rt-bfmr-link-sync-feedback-v2`; re-authored after a rejected first draft, refine loop still running 2026-10-06, nothing committed).
* **TLS fingerprint fix** (`rt-bfmr-tls-fingerprint`): see banner. Root causes found while authoring: the model's fixture imported a worktree-relative path and its fake HTTPS response never emitted `end`, hanging `loggedFetch` and freezing the queue lanes for ~16 min; harness guards (per-command wall timeout, network-denying preload, selfcheck-hang ledger signature) were added. Requeue still owed.
* Already on `main` for the order flags: "tracking not uploaded" honors reservation status (29f568b), "No reservation" chip suppressed on cancelled orders (2578a97), Submit not offered without tracker id (b68224c).

### Operational warning

The resell-tracker app's own `GET /api/settings` returns BFMR password/API secret **in plaintext** to any caller sending `Cookie: resell_uid=1`. Never call it while probing; a masking fix is owed. (This is the app's endpoint, not BFMR's.)

---

## Known Unknowns

| # | What | Why needed |
|---|------|------------|
| 1 | Exact `order_data` payload shape for `POST /api/orders` on `hr-ext-api.bfmr.com` | Rewrite `submitTracking()` to use TrackForMe API instead of `POST /api/my-tracker` |
| 2 | Authenticated 200 through `node:https` `ciphers: 'DEFAULT'` on `www.bfmr.com` (and whether writes work from server Node) | Decides whether the server can push order numbers/tracking itself or the sidecar must (bundle `rt-bfmr-tls-fingerprint`) |
| 3 | Whether the `POST /api/login` 403 is reCAPTCHA v3, the TLS fingerprint, or both | Would remove the manual session-seed step if it is only the TLS rule |
| 4 | Whether BFMR `search=<order number>` on `GET /api/v2/my-tracker` matches order numbers | The vanished-reservation lookup depends on it |
| 5 | Whether `POST /api/my-tracker` on a shipped row always replaces the reservation (new `my_tracker_id`) or only sometimes | Needed before wiring `supersededRowIds` / replace-tracking into the UI |

~~`GET /deal/reservations/active` response shape~~ — resolved 2026-08-23, see REST API Endpoints section.

---

## Integration Files

| File | Purpose |
|------|---------|
| `lib/bfmr.ts` | REST API client — `X-API-KEY`/`X-API-SECRET` auth, deals, reservations, shipments |
| `lib/bfmrWeb.ts` | Web App API client — JWT login, profile + extToken fetch, tracker fetch, tracking submission |
| `app/api/bfmr/web-login/route.ts` | POST handler — logs in with email/password, fetches and saves all 5 settings |
| `app/api/bfmr/full-sync/route.ts` | POST handler — bulk BFMR order sync, called by "Resync Groups" |
| `app/api/bfmr/sync-reservations/route.ts` | POST handler — REST status-enum pull by scope, vanished lookup, upserts keyed on `lineKey`, web backfill of `myTrackerId`, auto-link, stale-link reconciliation |
| `app/api/bfmr/web-session-seed/route.ts` | POST handler — stores a browser-captured session (token/xsrf/cookies/expiry) |
| `lib/bfmrSyncScope.ts`, `lib/bfmrSyncTrigger.ts` | Scope enums (13 all / 7 open statuses) and the trigger → scope policy |
| `lib/bfmrVanished.ts`, `lib/bfmrReservationLineKey.ts` | Vanished-order lookup, superseded-row detection, split-safe line key |
| `lib/bfmrJoin.ts` | Pure join of the REST and Web ID spaces (`myTrackerId` backfill, split-group matching) |
| `lib/bfmrLinkGuard.ts`, `lib/bfmrLinkReconcile.ts`, `lib/bfmrCrossOrderLinks.ts`, `lib/bfmrPushGate.ts` | Link invariants, canonical-link selection, cross-order exclusion, order-number push decision |
| `lib/bfmrSubmitFlow.ts`, `lib/bfmrVerify.ts`, `lib/bfmrReplaceTracking.ts` | Accept/verify decoupling, verify read-back, replace-tracking payload helpers (not wired) |
| `sidecar/src/bfmr.js` | Sidecar login confirm, header interceptor, in-page tracker fetch, `syncBfmr` |

**Settings keys:** `bfmr_email`, `bfmr_password`, `bfmr_api_key`, `bfmr_api_secret`, `bfmr_ext_token`, `bfmr_session_token`, `bfmr_session_xsrf`, `bfmr_session_cookies`, `bfmr_session_expires`

*Last verified against resell-tracker `main` (2f3293c) and the dispatch log on 2026-10-06.*


## Sync transport (2026-10-08)
Resync Groups gets BFMR tracker rows over the API (server-side sync-reservations, node:https default ciphers past the WAF, stored ~30-day web JWT). The sidecar browser (SYNC_BFMR) is a fallback only, for when that session is expired (BFMR login is reCAPTCHA-gated).
