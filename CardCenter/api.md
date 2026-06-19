# CardCenter API

Base URL: `https://cardcenter.cc`

## Authentication

POST `/Api/Tokens`

```json
{ "email": "...", "password": "..." }
```

Response field: `token` (also check `accessToken`, `access_token` as fallbacks).
Use as `Authorization: Bearer <token>` on all subsequent requests.

---

## Endpoints

### GET `/Api/Reservations`

Returns already-approved reservations waiting for card submission. Reservations are NOT created here — they're created as a side effect of `POST /Api/Rates/{id}/Actions/ReserveCap`.

Response shape: `{ items: CcReservation[] }` or `CcReservation[]` directly.

```ts
interface CcReservation {
  id: number;
  date: string;
  seller: { id: number; email: string };
  brand: CcBrand;
  buyOrder: { id: number; value: number; brand: CcBrand; buyer: { id: number; displayName: string; email: string } };
  value: number;
  quantity: number;
  rate: number;
  submissionTerms: number;
  paymentTerms: number;
  flexType: string;
  status: string;           // "Approved" = open, "Closed" = fulfilled
  expired: boolean;
  submissionDeadline: string;
  submissionToken: string;
  permissions: Record<string, boolean>;  // permissions.submit = false means blocked
}
```

Filter for submittable reservations: `status === 'Approved' && !expired && permissions.submit !== false`

---

### POST `/Api/Rates/{buyOrderId}/Actions/ReserveCap`

**Creates a reservation** for a buy order. This is how reservations are made — not via a separate reservations endpoint.

Request:
```json
{ "quantity": 4 }
```

Response: A submission object (same shape as `GET /Api/Submissions/{id}`) with a nested reservation. The submission UUID is returned immediately; the reservation starts in `Processing` status and transitions to `Approved` within a few seconds.

```ts
interface ReserveCapResponse {
  id: string;           // submission UUID — poll GET /Api/Submissions/{id}
  date: string;
  buyer: { id: number; displayName: string; email: string };
  seller: { id: number; email: string };
  readyForPayment: boolean;
  groups: [{
    id: string;
    brand: CcBrand;
    value: number;
    reservation: {
      id: number;
      status: "Processing" | "Approved";
      submissionToken: string;
      submissionDeadline: string;   // only present once Approved
      expired: boolean;
      permissions: { submit: boolean };
      // ...full CcReservation fields
    };
  }];
}
```

After calling this, poll `GET /Api/Submissions/{id}` until `groups[0].reservation.status === "Approved"`.

---

### GET `/Api/BuyOrders/v2?organization=&brand=&date=&pageSize=0`

Full marketplace of open buy orders — much wider pool than reservations. Each item has the same shape as `/Api/Rates/{id}` minus `sellerAgreement` and `submissionInstructions`.

---

### GET `/Api/Rates/{buyOrderId}`

Detail for a single buy order. Includes everything needed to submit a card without a separate agreement fetch.

```json
{
  "id": 26902,
  "deal": { "title": "...", "vendor": { "name": "Costco", "id": 4 }, "id": 729604 },
  "buyer": { "id": 1051, "displayName": "CardCenter", "email": "support@cardcenter.cc" },
  "brand": { "name": "DoorDash", "slug": "doordash", "type": "Standard", "image": { "id": "..." }, "id": 5950 },
  "value": 50,
  "rate": 0.845,
  "purchaseRate": 0.8,
  "margin": 0.053,
  "paymentTerms": 14,
  "maximumPaymentTerms": 60,
  "flexType": "None",
  "autoApproveOffersUntil": "2026-07-01T03:59:00Z",
  "maximumSubmissionTerms": 2,
  "unfulfilledPerSellerCap": 2000,
  "availableCap": 51750,
  "startDate": "2026-06-01T11:37:40Z",
  "submissionInstructions": {
    "defaultBrand": {
      "brand": { ... },
      "entryInstructions": "Submit only the 11 or 16 character alphanumeric code.",
      "pinValidation": "NotAllowed",
      "expirationDateValidation": "NotAllowed"
    },
    "defaultValue": 50
  },
  "sellerAgreement": {
    "id": "...",
    "date": "...",
    "agreement": {
      "id": "d969eab1-0f19-11f0-a716-0e01bba92d3d",
      "date": "2025-04-01T16:53:34Z"
    }
  },
  "permissions": { "submit": true }
}
```

`sellerAgreement.agreement` is the `acceptAgreement` object used in submissions — no separate fetch needed.

---

### POST `/Api/Reservations/{reservationId}/ParsedCards`

Validates card codes against a reservation. Called before submission to parse and verify codes.

Request:
```json
{ "text": "CODE1\nCODE2\nCODE3\nCODE4" }
```
(Newline-separated card codes)

Response:
```json
{
  "cards": [
    { "brand": CcBrand, "value": 50, "code": "CODE1" }
  ],
  "submission": {
    "groups": [
      {
        "brand": CcBrand,
        "value": 50,
        "quantity": 4,
        "offers": [ { "reservation": CcReservation, ... } ]
      }
    ]
  }
}
```

The `submission.groups` from this response is **not** passed directly — each group is missing the `reservation` field required by `POST /Api/Submissions`. Before submitting, inject `{ reservation: { id: reservationId } }` into each group:

```ts
// reservationDetail = full object from GET /Api/Reservations/{id}
// CardCenter requires Brand, Seller, and Quantity — { id } alone returns 400
const groups = parsed.submission.groups.map(g => ({ ...g, reservation: reservationDetail }));
```

---

### GET `/Api/Reservations/{id}`

Single reservation detail. Same shape as `GET /Api/Reservations` items plus `submissionInstructions` and `sellerAgreement`. Use `reservation.seller` for the submission's `seller` field.

---

### POST `/Api/Reservations/{id}/Actions/Cancel`

Cancels an open reservation. Request body: `{}`. Returns the reservation with `status: "Canceled"`.

```json
{ "status": "Canceled", "permissions": { "comment": true } }
```

---

### POST `/Api/Submissions`

Submit gift cards for sale against a reservation.

Flow: `POST /Api/Reservations/{id}/ParsedCards` → strip groups to `brand/value/quantity` → inject full reservation → `POST /Api/Submissions`.

**IMPORTANT**: Do NOT spread the full ParsedCards group object. The `offers` field must be omitted — sending it causes a 400 `"Submissions must have at least one new reservation or at least one submitted card"`. Only send `brand`, `value`, `quantity`, and `reservation`.

Request payload:
```json
{
  "seller": { "id": number, "email": string },
  "groups": [
    {
      "brand": CcBrand,
      "value": number,
      "quantity": number,
      "reservation": CcReservation
    }
  ]
}
```

`seller` and `reservation` come from `GET /Api/Reservations/{id}`.
No `acceptAgreement` needed in this flow.

Response: submission UUID (`id`), `readyForPayment: boolean`, and groups. Reservation status transitions to `Closed` on success.

The POST response groups do NOT include `submittedCards`. Fetch `GET /Api/Submissions/{id}` immediately after to get per-card data including `giftCard.id` and `paymentReceivedOn`.

---

### GET `/Api/Payments`

Query params: `paidTo`, `paidBy`, `status`, `batch`, `purchaseOrder`, `recipientReconciled`, `senderReconciled`, `date.start`, `date.end`

**`paidTo` is required** — omitting it returns no results. Set it to the seller's numeric ID (e.g. `1056`). Resolve the seller ID from `GET /Api/Reservations` → `items[0].seller.id`.

Payment statuses (response `status` field): `Waiting` → `Sent` → `Completed`

API filter values (`status` query param) differ from response names:

| Response status | Filter param value |
|---|---|
| `Waiting` | `Scheduled` |
| `Sent` | `Sent` |
| `Completed` | `Completed` |

- `Waiting` / `Scheduled`: queued, no `id` yet
- `Sent`: in transit, has `id`
- `Completed`: received, has `id`

**Note:** With no `status` filter the API does not return all statuses — fetch each status separately and combine.

Response: `{ items: Payment[], nextPageToken: string }` (paginated)

```ts
interface Payment {
  $type: "Reseller";
  id?: number;                    // present on Sent/Completed
  name: string;                   // e.g. "P1056-20260625"
  amount: number;
  paymentMethod: "ACH";
  status: "Waiting" | "Sent" | "Completed";
  date: string;                   // when initiated
  receivedOn: string;             // expected/actual receipt date (YYYY-MM-DD)
  paidBy: { id: number; displayName: string; email: string };
  paidTo: { id: number; email: string };
  paidInto?: { ... };             // bank account detail, present on Sent/Completed
  senderReconciled: boolean;
  recipientReconciled: boolean;
  listings?: PaymentListing[];    // present when fetching individual payment
  permissions: Record<string, boolean>;
}
```

---

### GET `/Api/Submissions/{id}`

Full submission detail. Includes `submittedCards` per group. Available immediately after `POST /Api/Submissions`.

```ts
interface SubmittedCard {
  id: number;                      // listing ID (same as PaymentListing.listing.id)
  giftCard: { id: number; code: string };  // code is truncated: "…2SVV" (last 4 chars only)
  paymentDueDate: string;
  paymentSentOn: string;
  paymentReceivedOn: string;       // use as Order.overdueAt — no need to query /Api/Payments
  purchasePrice: number;
  value: number;
  brand: CcBrand;
}
```

`giftCard.id` = `GiftCard.ccGiftCardId`. Match to local cards by `cardNumber.endsWith(code.replace(/^…/, ''))`.

---

### GET `/Api/Payments/{id}`

Single payment detail. Includes `listings` array linking the payment to individual card submissions.

```ts
interface PaymentListing {
  id: number;
  amount: number;
  listing: {
    $type: "SellOffer";
    id: number;
    giftCard: { id: number };     // CardCenter's internal gift card ID — matches GiftCard.ccGiftCardId
    value: number;
    brand: CcBrand;
    purchasePrice: number;
    purchasePaid: number;
    paymentDueDate: string;
    paymentSentOn: string;
    paymentReceivedOn: string;
    createdAt: string;
    purchasedAt: string;
    purchasedBy: { id: number; displayName: string; email: string };
    permissions: Record<string, boolean>;
  };
}
```

`listing.giftCard.id` matches `GiftCard.ccGiftCardId`.

---

## Known Unknowns

| # | What | Why needed |
|---|------|------------|
| 1 | ~~Whether reservations can be created via API~~ **RESOLVED**: `POST /Api/Rates/{id}/Actions/ReserveCap` with `{"quantity": N}` | - |
| 2 | ~~Actual card submission endpoint and payload after reservation is Approved~~ **RESOLVED**: `POST /Api/Reservations/{id}/ParsedCards` → `POST /Api/Submissions` with `seller` + `parsed.submission.groups` | - |
| 3 | POST `/Api/Submissions` response body | Determine if CardCenter returns `giftCard.id` per card for payment matching — spy capture was truncated |

---

## Current Integration

| File | Purpose |
|------|---------|
| `lib/cardcenter.ts` | `getCcToken()`, `getPaymentDetail()`, `submitCards()` |
| `app/api/cardcenter/buy-orders/route.ts` | GET handler — all open buy orders grouped by brand (powers Rates page) |
| `app/api/cardcenter/rates/route.ts` | GET handler — buy orders filtered by brand+value (used by gift card panel) |
| `app/api/cardcenter/reserve/route.ts` | POST handler — reserve + submit in one step (from order detail gift card panel) |
| `app/api/cardcenter/reserve-only/route.ts` | POST handler — reserve without submitting cards (from Rates page) |
| `app/api/cardcenter/reservations/route.ts` | GET handler — open CC reservations, filterable by brand+value |
| `app/api/cardcenter/reservations/[id]/route.ts` | DELETE handler — cancels a reservation via Actions/Cancel |
| `app/api/cardcenter/fulfill-reservation/route.ts` | POST handler — submits cards to an existing reservation |
| `app/api/cardcenter/submit/route.ts` | POST handler — submits unsubmitted cards for an order (legacy, uses submitCards()) |
| `app/api/cardcenter/sync-payments/route.ts` | POST handler — bulk sync: matches gift cards by `ccGiftCardId` to payment listings, updates `bgPaidAmount` per order |
| `app/api/cardcenter/sync-payment/route.ts` | POST handler — single payment sync |
| `app/api/cardcenter/payments/route.ts` | GET handler — fetches payments (all statuses) with optional `status` filter |
| `app/api/cardcenter/brands/route.ts` | GET handler — returns unique brand names for autocomplete |

Credentials stored in `Setting` table as `cc_email` and `cc_password` per user.

Seller ID is resolved at runtime from `GET /Api/Reservations/{id}` → `reservation.seller.id` and used as `paidTo` on all payment requests.

`GiftCard.ccGiftCardId` stores CardCenter's internal gift card ID for payment matching. Currently populated via manual entry in the UI; auto-population on submission pending capture of the POST `/Api/Submissions` response (see Known Unknowns #3).
