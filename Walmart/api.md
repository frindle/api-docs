# Walmart

Walmart's order history is a React SPA — fetching the URL directly returns a shell with no order data. All scraping is done from the live DOM by a browser extension content script.

## Pages

### Order List

**URL:** `https://www.walmart.com/orders`

Orders are rendered into the DOM as React components. There is no parseable JSON on the list page — detail pages must be fetched individually.

**DOM selectors (try in order):**
1. `[data-testid*="orderGroup"]`
2. `[data-testid*="order-card"]`
3. `[data-testid*="orderCard"]`

**Order number extraction:**
- From element with `id` matching `^caption-`
- Or text regex fallback

---

### Order Detail

**URL pattern:** `https://www.walmart.com/orders/{orderId}`

Order detail data is embedded in `<script id="__NEXT_DATA__">` as JSON.

**JSON path to order:** `props.pageProps.initialData.data.order`

**Key fields:**

| Path | Description |
|------|-------------|
| `order.priceDetails.grandTotal.value` | Order total in USD |
| `order[groups_{version}][0]` | First shipment group (version may vary) |
| `firstGroup.deliveryAddress.address.addressString` | Delivery address |
| `firstGroup.items[0].productInfo.name` | First item name |

**Note:** The `groups` key name varies (e.g., `groups_v2`, `groups_v3`). Match via `Object.keys(order).find(k => k.startsWith('groups_'))`.

## Delivery photo

When the carrier captures a proof-of-delivery image, the order detail page renders:

- Trigger button (always present on delivered orders): `button[data-automation-id="view-photo-link"]` → "View delivery photo".
- Image element: `img[alt="Proof of delivery location"]` (also matched by `img[src*="/delivery-photo/"]`).
- URL: signed proxy at `https://receipts-query.edge.walmart.com/delivery-photo/<base64-id>.jpeg`.
- TTL: signed and expiring (similar to Amazon's S3 link, exact lifetime not measured). Download server-side immediately rather than storing the URL.

The image is usually present in the SSR HTML alongside the button. If not, the modal-trigger button is present but the `<img>` is rendered only after the user clicks — in that case the static fetch won't capture it.

## Sync Behavior

- Syncs back 30 days, or to `walmartLastSync - 48 hours`, whichever is earlier
- Fetches up to 3 detail pages concurrently

**⚠ Known bug (confirmed 2026-09-23, not yet fixed): `isStoreDelivery` computed but never gates the tracking fallback**

`sidecar/src/walmart.js` computes `isStoreDelivery` from the order-detail HTML (`/Delivery\s+from\s+store/i.test(html)`, `walmart.js:251`) and returns it on the detail object (`walmart.js:357`), but nothing ever reads it before `syncWalmart`'s digits-only fallback fires:

```js
// walmart.js:442-444
const filteredTracking = detail.tracking.filter(t => t !== order.orderNumber);
if (filteredTracking.length) order.trackingNumbers = filteredTracking;
else order.trackingNumbers = [order.orderNumber.replace(/[^0-9]/g, '')];
```

When no real tracking number was scraped, this unconditionally falls back to the digits of the order number itself as a fake "tracking number" — regardless of whether the order is genuinely store-pickup/store-delivery (where an order-number-as-tracking placeholder is reasonable) or simply an in-transit shipped order that hasn't had its tracking number rendered/scraped yet (where this fabricates a bogus tracking number that will never resolve). `isStoreDelivery` exists specifically to make that distinction and is silently discarded. Documenting as a known gap, not fixing here.

## Integration Files

| File | Purpose |
|------|---------|
| `src/content/walmart.ts` | Browser extension content script — order list DOM scrape, `__NEXT_DATA__` detail parse |
