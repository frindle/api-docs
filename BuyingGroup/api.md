# BuyingGroup API

**Base URL (data):** `https://api.prod.buyinggroup.com/v1`
**Base URL (auth):** `https://api.prod.buyinggroup.com/v2`

> Last verified against resell-tracker `main` (`lib/buyinggroup.ts`, `lib/bgSync.ts`, `lib/bgAuth.ts`, `lib/bgReceiptAttribution.ts`) on 2026-10-06. Sections below marked *(corrected 2026-10-06)* replace earlier claims that did not match the client code.

**Transport behaviour:** BG's edge sporadically returns 502/503/504 on healthy calls; the client retries **once** after 750 ms on those statuses (and on a network `TypeError`), for data calls and for `add_trackings`. All calls go through `loggedFetch` (group `BuyingGroup`).

## Authentication

Django SimpleJWT. Login returns tokens nested under a `token` key (`extractTokens` accepts `access`/`refresh` or `auth_token`/`refresh_token`; BG's own site stores the access token as `auth_token` in localStorage).

**How resell-tracker gets an access token (`lib/bgAuth.ts`, corrected 2026-10-06):** refresh-first. If a stored `bg_refresh_token` exists it calls `/v2/token/refresh`; only if that fails does it log in with `bg_email`/`bg_password`, and after a login it immediately exchanges the returned refresh token for an access token (the code treats login as yielding only a usable refresh token). Settings keys: `bg_access_token`, `bg_refresh_token`, `bg_email`, `bg_password`, `bg_sync_start_date`.

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

List payments. *(Corrected 2026-10-06: paginated, and the list lives under `payload.payments`.)*

**Request:** `{ "page": 1, "page_size": 100 }` — the old bare `{}` only returned the server's default first page, so `REQUESTED` payments beyond it were silently dropped and not-yet-disbursed receipts were treated as truly paid. The client pages (up to 50 pages) until a short page, and also stops if a page adds no new `payment_id` (in case the endpoint ignores paging params).

**Response:** `{ "payload": { "payments": [ ... ] } }`. Payment fields used: `key`, `payment_id`, `amount` (string), `type`, `status`, `paid_dt`, `processed_dt`, `created_dt`.

**Payment statuses:** `"PAID"`, `"REQUESTED"`, `"SENT"`. The sync logs a status breakdown each run because any other status (e.g. a `PROCESSING`) would be missed by the `REQUESTED` filter and receipts would read as paid before the money lands.

---

### Receipts

#### `POST /v1/receipt/get_receipts`

List receipts (paginated).

**Request:**
```json
{ "page": 1, "page_size": 50 }
```

**Response (corrected 2026-10-06):** list is at `payload.receipts` (the client also tolerates a bare array, `results` or `data`). Fields the sync reads: `receipt_id`, `order_number`, `status` (e.g. `verified`; `return`/`returned`/`refund`/`refunded` are treated as returns), `paid` (boolean), `total`, `total_paid`, `created_dt`, `modified_dt`, nested `tracking.tracking_id`. The sync pages until a page has fewer than 50 items.

**How the tracker reads a receipt (`lib/bgReceiptAttribution.ts`, `lib/bgSync.ts`):**
* *In balance* = not a return and (`paid === true` or `status === "verified"`). *Truly paid* = `paid === true` and not held back as "credited only".
* "Credited only" heuristic: sum the `REQUESTED` payments, then walk the newest `paid` receipts and hold back receipts (newest first) while their `total_paid`/`total` still fits that requested amount — money credited to balance but not yet disbursed.
* Receipts are matched to local orders by digits-only `order_number`, with a one-trailing-digit-dropped fuzzy fallback, then by tracking number (a tracking shared by several orders splits the receipt total proportionally to `bgExpectedPayout`/`salePrice`). Receipts created before `bg_sync_start_date` are ignored. Orders whose buyer name matches "Card Center" are excluded from attribution entirely (commit 5e5cd83, 2026-10-02).
* An order is `bgCredited` only when **every** one of its distinct shipment trackings has an in-balance receipt (`lib/bgCredited.ts`; commit 4032e6f — the old first-receipt rule showed order 899 as credited with one of two packages). A BG receipt for an order assigned to a BFMR buyer sets `buyerMismatch`. A BG order with no receipt after 14 days (Amazon/Walmart only) is flagged overdue.

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

**Response (corrected 2026-10-06):** list at `payload.orders` (bare array / `results` / `data` also tolerated). The sync only needs the nested `tracking.tracking_id` (or `tracking_id`) of each order to learn which tracking numbers have already been submitted to BG, and then sets `trackingSubmittedToBg` on matching local orders (digits-only comparison). It pages until a page has fewer than 50 items.

---

#### `POST /v1/order/add_trackings`

Submit tracking numbers. Uses multipart `FormData` (no JSON `Content-Type`; only the bearer header) — the `tracking_list` field is a JSON-encoded array. Retried once on 502/503/504.

**FormData field:**

| Field | Type | Value |
|-------|------|-------|
| `tracking_list` | string (JSON) | `["1Z..."]` as sent by `lib/buyinggroup.ts` `submitTracking(token, trackingNumbers: string[])` |

*Discrepancy to verify:* an earlier version of this doc showed objects (`[{"tracking_number": "1Z...", "order_id": 123}]`). The current client sends a plain array of strings; which shape(s) BG accepts has not been re-confirmed.

---

### Deals

#### `GET /v1/deal/get_deals_new`

List deals.

**Query params** (as sent by `getDeals`):

| Param | Description |
|-------|-------------|
| `page` | Page number |
| `page_size` | Items per page (client default 60) |
| `data_type` | One of `on_sale_now`, `below_cost`, `all`, `active` (client default `active`) |
| `buyer_view` | Set `true` for buyer-facing view |
| `is_full`, `is_locked` | Sent as the literal string `null` |
| `title` | Title search (empty string = none) |

**Response:** `{ "payload": { "deals": [...], "count": N } }` — deals are under `payload.deals`, not `payload.results`. Deal fields used: `key`, `deal_id` (e.g. `DL-06260061`), `title`, `active`, `price` (what you pay at the store), `commission` (what BG pays you), `old_price`, `commit_required`, `commit_locked`, `is_special`, `expiry_day` (`MM-DD-YYYY`), `flags[]`, `deal_item[].item_stores[]` (store links).

---

### Dashboard

#### `POST /v1/dashboard/get_statistics`

Get summary statistics for the dashboard.

**Request:** `{}`

**Response:** `payload.balance.remaining_balance` is the BG account balance (`getBalance` parses it as a float).

---

### Commitments

#### `POST /v1/commitment/get_commitments`

Lists active (and possibly historical) commitments for the authenticated buyer.

**Request:** `{ "page": 1, "page_size": 100 }` *(corrected 2026-10-06: it paginates; the original capture was an empty body)*. Response: `payload.commitments[]` plus `payload.count`; `POST /api/buyinggroup/sync-commitments` loops pages of 100 (safety cap 50 pages) until it has `count` rows, then upserts by `(userId, commitmentId)`.

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
| `status` | See status values below |
| `tracking_linked_required` | If true, BG expects tracking link before counting fulfilled |

**Status values** (confirmed 2026-06-26):

| Value | Meaning | Has open slots? |
|-------|---------|-----------------|
| `ACTIVE` | Open, no slots shipped yet | Yes (if `count > assigned`) |
| `PARTIALLY FULFILLED` | At least one slot shipped, others still open | Yes (if `count > assigned`) |
| `FULFILLED` | All slots delivered | No |
| `VOIDED` | BG cancelled the commitment (deal expired, removed, etc.) | No, `count` is set to `0` |

**Gotcha:** BG flips a commitment to `PARTIALLY FULFILLED` as soon as the FIRST slot ships, even when 4 of 10 slots are still open and linkable. Don't filter on `status === 'ACTIVE'` alone — use `OPEN_STATUSES = { ACTIVE, PARTIALLY FULFILLED }` or `remaining > 0`.

---

### Commitment mutations (extension API spy)

Endpoints that the BG site's UI hits when the user edits commitment state. Our extension API spy watches for successful (2xx) responses on these and triggers `/api/buyinggroup/sync-commitments` so local state stays fresh without manual "Sync from BG" clicks.

| Endpoint | When fired |
|----------|------------|
| `POST /v1/commitment/edit_commitment` | User raises/lowers `count` on an existing commitment |
| `POST /v1/commitment/create_commitment` | User opts in to a new deal |
| `POST /v1/commitment/delete_commitment` | User removes a commitment |
| `POST /v1/commitment/update_commitment` | Generic update (used by some flows) |
| `POST /v1/commitment/cancel_commitment` | User cancels a commitment |

resell-tracker's own server also calls three commitment endpoints directly with the bearer token (`lib/buyinggroup.ts`):

| Endpoint | Request body | Purpose |
|----------|--------------|---------|
| `POST /v1/commitment/get_commitment_items` | `{ "deal_key": "<uuid>" }` | List a deal's selectable items (`payload.commitment_items[]`: `key`, `item_title`, `item_model`, `item_id`, `upc`, `limit_total`, `limit_user`, `enabled`, `in_stock`, `commission`, `current_cost`, `total`, `bg_points_reward`, `commitment_count`, `commitment_percentage`) |
| `POST /v1/commitment/save_commitment` | `{ "deal_key": "...", "item_key": "..." }` | Create a commitment on an item (note: the site-side spy name is `create_commitment`; the client uses `save_commitment` — which of the two the site UI itself calls is unverified) |
| `POST /v1/commitment/edit_commitment` | `{ "deal_key": "...", "item_key": "...", "quantity": N }` | Change a commitment's quantity |

The spy debounces these by ~5s so a rapid drag-edit (e.g. `3 → 6 → 3` in 2 seconds) coalesces into one sync. Auth is the BG site's own cookie session; we don't call these directly from the server.

---

### Notifications

#### `GET /v1/notification/get_buyer_notification_count`

Lightweight polling endpoint for unread notification count.

**Response:** `{ "status": "SUCCESS", "payload": { "notification_count": 0 } }`

## Integration Files

| File | Purpose |
|------|---------|
| `lib/buyinggroup.ts` | API client — auth, orders, receipts, payments, deals, commitments, balance, tracking submit |
| `lib/bgAuth.ts` | Refresh-first access-token helper, `isBgConfigured` |
| `lib/bgSync.ts` | `runBgReceiptSync`: payments + receipts + orders reconcile (paid/credited/overdue/`trackingSubmittedToBg`/`buyerMismatch`) |
| `lib/bgReceiptAttribution.ts`, `lib/bgCredited.ts` | Pure receipt → order attribution (CardCenter excluded) and the all-shipments-credited rule |
| `app/api/buyinggroup/*` | Routes: `login`, `token`, `deals`, `deals/history`, `commitments`, `commitment` (+`list`, `items`), `sync-commitments`, `links` (+`[id]`), `resolve-order`, `order-payouts`, `orders`, `pending-orders`, `receipts`, `sync-orders`, `tracking-orders` (normalized tracking/order number → BG- or BFMR-buyer orders), `debug` |

Linking a BG commitment to an order auto-sets the order's `deliveryDeadline` from the commitment's `expiry_day` (commit d1bd037); BG links are guarded against duplicate-tracking and over-allocation (commit 176225d, order-907 class).

**Not landed (do not document as behaviour):** hiding EXPIRED commitments from the linking list (`rt-bg-commitments-hide-expired`, still in the queue 2026-10-06).


## Paging (2026-10-08)
List endpoints (get_receipts, get_orders, get_payments, get_commitments) read `page`/`page_size` from **multipart/form-data**, as the web app sends them. A JSON body's paging is ignored (returns page 1, 25 rows). resell-tracker uses `bgListForm` (lib/bgPaging.ts).
