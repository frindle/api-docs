# Amazon

Amazon has no public API for order history. Scraping is done via a browser extension content script using a mix of DOM scraping and background-worker fetch (to avoid browser trust-token quota limits).

## Pages

### Order List

**URL pattern:** `https://www.amazon.com/your-orders/orders?startIndex={n}&timeFilter=year-{YYYY}`

Paginated in steps of 10. Sync window: 60 days back, or to the stored `amazonLastSync` timestamp, whichever is earlier. Without `timeFilter`, the page only returns the past ~30 days regardless of what the user wants. For a sync spanning multiple calendar years, iterate one year at a time using `timeFilter=year-YYYY`.

**Critical rendering quirk** (confirmed 2026-06-28 across v1.1.61 → v1.1.68 of the extension):

- **Current year:** The page is fully React-rendered. Any **static `fetch()`** of `?startIndex=N&timeFilter=year-{currentYear}` returns the SPA shell (no order links). Tested both background-SW (cross-origin) and content-script (same-origin, `Sec-Fetch-Site: same-origin`) — neither populates. Only full JS execution renders the order cards.
- **Past years:** Server-rendered. Static `fetch()` with `timeFilter=year-{pastYear}` returns full HTML with order cards. This is the easy/cheap path.

**Working scrape strategy:**
1. Current year → walk the live tab itself. Scrape the live DOM, persist resume state (`chrome.storage.local` + `sessionStorage`), `location.href = '/your-orders/orders?startIndex={N}&timeFilter=year-{currentYear}'`. New content-script instance auto-attaches on the next page, reads resume state, repeats.
2. Past years → `fetch()` based loop, same code path that's worked since v1.1.61.

**Pagination markup** lazy-injects after the order cards render — wait ~1.5s after `waitForOrders()` before reading the Next link. Markup also changes periodically (mid-2026 dropped `.a-pagination .a-last` for `ref_=ppx_yo2ov_dt_b_pagination_*`). Use a text-based fallback (`<a>` text starts with "Next" or contains "→") for resilience.

---

### Order Detail

**URL pattern:** `https://www.amazon.com/gp/your-account/order-details?orderID={orderId}`

Fetched via background worker. Contains ship-track links used to find tracking numbers.

---

### Tracking pages (legacy + new UI)

Amazon serves tracking through two different page layouts. We follow either.

**Legacy:** `a[href*="ship-track"]` links on the order detail page. One sub-page per shipment.

**New "package tracker" UI** (rolled out 2026):
- Link selectors widened to also include `a[href*="progress-tracker"]` and `a[href*="package-tracking"]`.
- The tracking number is rendered inline in a `<div class="pt-delivery-card-trackingId">Tracking ID: …</div>`. Sometimes there's NO sub-link to follow — the modal trigger ("See all updates") uses `href="#"`. To handle this:
  - Explicit hook: `doc.querySelector('.pt-delivery-card-trackingId, [class*="trackingId"]')`.
  - Also scan the order **detail page** itself (not just sub-pages) — pt-cards sometimes inline the tracking ID right there.

Per-order cap is **8 tracking pages** (split shipments routinely exceed 3).

## Tracking Number Patterns

Extracted via regex from page HTML:

| Carrier | Pattern |
|---------|---------|
| AMZL (Amazon Logistics) | `TBA` followed by 12–15 digits |
| UPS | `1Z[A-Z0-9]{16}` |
| USPS | `9[0-9]{19,21}` |
| FedEx | `[1-8][0-9]{14}` |

**Cleanup gotcha:** Some pages have trailing UI text glued to the tracking number ("TBA123See"). Strip trailing `[A-Za-z]+$` — **but skip UPS numbers** (`/^1Z/i.test()`) because UPS 1Z codes legitimately end in `[A-Z]` and stripping corrupts them.

**Carrier link fallback:** When the tracking number doesn't appear in page text, parse it out of any carrier-tracking link param: `qtc_tLabels1|tLabels|tracknum|InquiryNumber\d*|tracknumbers|trknbr|AWB|tracking[_-]?number[s]?|trackingNumber`.

---

### Delivery photo

When the carrier captures a proof-of-delivery image:

- Selector: `img.photo-on-delivery-img-thumb` (also `[class*="photo-on-delivery"]`).
- URL: signed S3 link at `https://us-prod-temp.s3.amazonaws.com/imageId-<uuid>?X-Amz-Algorithm=…&X-Amz-Expires=259200&…`.
- TTL: **3 days** (`X-Amz-Expires=259200`). Download server-side immediately; don't store the URL.
- Sometimes the `src` is set; sometimes only `data-src` (lazy load). Read `data-src` first, then `src`.

---

### Not-found / inaccessible orders

Amazon Business orders use the `113-` order-number prefix and aren't accessible from a personal session — the detail page returns "We can't find an order with that number." Detect with: `/We can't find an order with that number|Looking for an order|Page Not Found/i.test(detailHtml)` AND `!/order-details|order-summary|orderDetails|pmts-payments/i.test(detailHtml)`. Drop the row rather than pushing an empty order.

---

### No-Rush delivery bonus

Amazon's "Earns extra N% on items using No-Rush delivery" disclosure appears in the supplemental payment block under `#pmts-payments-instrument-supplemental-box-paystationpaymentmethod`. Regex: `/(?:extra|additional)\s+(\d+(?:\.\d+)?)\s*%[^<]{0,80}No[- ]?Rush/i`.

## Payment (card last-4 via iris iframe)

Amazon moved the charged payment method off the order-detail main document and into a cross-origin **`iris.apx.amazon.dev`** iframe (a Next.js frame). The main-document scrape can no longer see the card, so the sidecar (Playwright) traverses frames to recover the last-4. This is confirmed against `sidecar/src/amazon.js` (the server-side Playwright sidecar, not the `src/content/amazon.ts` extension path).

### Where it appears

On the order-detail page (`/gp/your-account/order-details?orderID=…`). The main document still exposes the payment box under `[class*="paystationpaymentmethod"]`, and `extractDetailInBrowser` scrapes a last-4 from it first (`amazon.js:339-351`) using, in order: `ending in NNNN`, `ending NNNN`, `**NNNN`, `xxxxNNNN`, `••••NNNN`. It scopes to **all** matching payment boxes (not just the first) because gift-card balance / promo / split-payment lines elsewhere on the page also contain "ending in ####" and used to win first-match. When that main-document scrape yields nothing, the order's real card now lives only in the iris iframe.

### Frame traversal + extraction

In `fetchOrderDetails` (`sidecar/src/amazon.js:511-523`), the iris fallback fires **only when `detail.paymentLast4` was not already set** by the main-document scrape:

```js
if (!detail.paymentLast4) {
  const irisFrame = page.frames().find(f => /iris\.apx\.amazon\.dev/.test(f.url()));
  if (irisFrame) {
    const frameText = await irisFrame.evaluate(() => document.documentElement.outerHTML).catch(() => null);
    const last4 = extractIrisLastDigits(frameText);
    if (last4) detail.paymentLast4 = last4;
  }
}
```

`extractIrisLastDigits(rawText)` (`sidecar/src/amazon.js:738-747`) is a pure/total parser (any non-string/empty/malformed input returns `null`, never throws):

1. **Structured first** — matches the frame's `__NEXT_DATA__`: `"paymentMethodNumber": { … "lastDigits": "NNNN" }` and returns those 4 digits.
2. **Masked visible-text fallback** — if no structured match, tries `••••NNNN` (`[•·․⋅●]{2,}`), `**NNNN`, `xxxxNNNN`, then `ending in NNNN`.

It deliberately keys on those mask/anchor patterns so decoy 4-digit runs (expiry year, order id, dollar amount) are never returned. Exported from the module (`amazon.js:751`) and unit-tested separately from the live-frame integration.

### Rate scrape (tier disambiguation)

Alongside the last-4, `extractDetailInBrowser` scrapes the payment box's own **"Earns X% back[, and extra Y% on …]"** text into `paymentRatePercent` (`amazon.js:359-365`) — base rate plus every `extra N%` summed. This is the only reliable signal for picking the right saved card when several share one physical last-4 at different bonus tiers (e.g. Amazon Store Card base / No-Rush / Amazon Day).

### `paymentLast4` → `cardId` mapping (server import)

The sidecar sends `paymentLast4` (and `paymentRatePercent`) per row to `POST /api/import`. `resolveCardId(row)` (`app/api/import/route.ts:215-238`) maps it to a saved card:

- **Explicit `cardId` on the row wins.** Otherwise, build a `last4 → card` map from the user's `creditCard` rows, including the `lastFours` relation for additional numbers (`import/route.ts:168-209`).
- If exactly one saved card has that last-4 → auto-assign it.
- If two cards **share** the last-4 (ambiguous), auto-assign is skipped **unless** the scraped `paymentRatePercent` matches exactly one candidate's `rewardsRate`, which resolves the tie (`import/route.ts:223-229`).
- On **create**, the resolved id is written as `cardId` (`import/route.ts:275`). On **update**, it is `existing.cardId ?? resolveCardId(r)` (`import/route.ts:322`) — it only **fills an unset** `cardId`, never overwrites an existing assignment.

### PATCH `cardId` on an existing order (new)

`cardId` is now a patchable field on `PATCH /api/orders/{id}`: it was added to `PATCHABLE_FIELDS` (`app/api/orders/[id]/route.ts:191`), and the handler coerces it — a numeric string becomes an `int`, and `null`/`''` unassigns — so the raw body value never reaches Prisma as a string (`route.ts:234-236`). `cardId` is also settable via the full-form `PUT` (`route.ts:97`). The order lock still blocks a `cardId` PATCH (only a `cashbackAmount`-only correction is lock-exempt).

## Sync Behavior

- Looks back 60 days or to `amazonLastSync`, whichever gives a shorter window
- Iterates order list pages until all orders in window are processed
- Fetches detail page + up to 3 ship-track pages per order
- On each detail fetch, recovers the charged card last-4 (main document, then the iris iframe fallback) and, server-side at import, resolves it to a saved `cardId`

## Integration Files

| File | Purpose |
|------|---------|
| `src/content/amazon.ts` | Browser extension content script — order list, detail fetch, tracking extraction |
| `sidecar/src/amazon.js` | Playwright sidecar — order-list walk, detail fetch, iris-iframe last-4 (`extractIrisLastDigits`, `fetchOrderDetails`), tracking + delivery-photo extraction |
| `app/api/import/route.ts` | Server import endpoint — `resolveCardId` maps scraped `paymentLast4`/`paymentRatePercent` to a saved `cardId` |
| `app/api/orders/[id]/route.ts` | Order CRUD — `PATCH`/`PUT` allow setting `cardId` on an existing order |
