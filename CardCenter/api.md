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

Pre-assigned reservations for your account. Narrower pool than buy orders — only reservations CardCenter has explicitly allocated to you.

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

### POST `/Api/PotentialSubmissions`

Alternative way to get the `acceptAgreement` object (used in current implementation as fallback).

Request: `{ "cards": [] }`

Response: `{ sellerAgreement: { agreement: { id: string, date: string } } }`

> **Note:** Use GET `/Api/Rates/{id}` instead when possible — the agreement is already embedded there.

---

### POST `/Api/Submissions`

Submit gift cards for sale.

#### Current implementation (reservation-based)

Matches cards against `/Api/Reservations` by `brand.name` (case-insensitive) + `value`.

Request payload:
```json
{
  "seller": { "id": number, "email": string },
  "acceptAgreement": { "id": string, "date": string },
  "groups": [
    {
      "brand": { "name": string, "slug": string, "type": string, "image": { "id": string }, "id": number },
      "cards": [
        {
          "brand": { ... },
          "code": "CARDNUMBERHERE",
          "value": 100,
          "quantity": 1,
          "reservation": { ...full CcReservation object... }
        }
      ]
    }
  ]
}
```

Cards are grouped by `brand.id`. The `reservation` object is the full reservation from GET `/Api/Reservations`, sorted by soonest `submissionDeadline`. Each reservation is used at most once.

#### TODO: buy-order-based submission

When submitting against a buy order directly (no pre-assigned reservation), the `reservation` field in each card may become `buyOrder` or `rate` — **unknown, needs capture**. See task: "Capture POST /Api/Submissions response from CardCenter site".

Response: **unknown** — need to capture to determine if CardCenter returns `giftCard.id` per submitted card (required for payment matching).

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

### GET `/Api/Payments/{id}`

Single payment detail. Includes `listings` array linking the payment to individual card submissions.

```ts
interface PaymentListing {
  id: number;
  amount: number;
  listing: {
    $type: "SellOffer";
    id: number;
    giftCard: { id: number };     // CardCenter's internal gift card ID — use for matching
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

`listing.giftCard.id` links to our `GiftCard.ccGiftCardId` column (TODO: add this column and populate on submission).

---

## Known Unknowns

| # | What | Why needed |
|---|------|------------|
| 1 | POST `/Api/Submissions` request payload when using a buy order (not reservation) | Rewrite `submitCards()` to use `/Api/BuyOrders/v2` instead of `/Api/Reservations` |
| 2 | POST `/Api/Submissions` response body | Determine if CardCenter returns `giftCard.id` per card for payment matching |
| 3 | Whether reservations can be created via API | Automation of reservation step |

---

## Current Integration

| File | Purpose |
|------|---------|
| `lib/cardcenter.ts` | `getCcToken()`, `getPaymentDetail()`, `submitCards()` |
| `app/api/cardcenter/submit/route.ts` | POST handler — submits unsubmitted cards for an order |
| `app/api/cardcenter/sync-payments/route.ts` | POST handler — bulk sync: matches gift cards by `ccGiftCardId` to payment listings, updates `bgPaidAmount` per order |
| `app/api/cardcenter/sync-payment/route.ts` | POST handler — single payment sync |
| `app/api/cardcenter/payments/route.ts` | GET handler — fetches payments (all statuses) with optional `status` filter |
| `app/api/cardcenter/test/route.ts` | GET handler — verifies auth, reservations, and agreement |
| `app/api/cardcenter/brands/route.ts` | GET handler — returns unique brand names for autocomplete |

Credentials stored in `Setting` table as `cc_email` and `cc_password` per user.

Seller ID is resolved at runtime from `GET /Api/Reservations` → `items[0].seller.id` and used as `paidTo` on all payment requests.

`GiftCard.ccGiftCardId` stores CardCenter's internal gift card ID for payment matching. Currently populated via manual entry in the UI; auto-population on submission pending capture of the POST `/Api/Submissions` response (see Known Unknowns #2).
