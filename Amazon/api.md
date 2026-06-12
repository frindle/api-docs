# Amazon

Amazon has no public API for order history. Scraping is done via a browser extension content script using a mix of DOM scraping and background-worker fetch (to avoid browser trust-token quota limits).

## Pages

### Order List

**URL pattern:** `https://www.amazon.com/your-orders/orders?startIndex={n}`

Paginated in steps of 10. Sync window: 60 days back, or to the stored `amazonLastSync` timestamp, whichever is earlier.

---

### Order Detail

**URL pattern:** `https://www.amazon.com/gp/your-account/order-details?orderID={orderId}`

Fetched via background worker. Contains ship-track links used to find tracking numbers.

---

### Ship-Track Pages

**URL pattern:** Extracted from `a[href*="ship-track"]` links on the order detail page.

Up to 3 ship-track pages are fetched per order with a 600ms delay between each.

## Tracking Number Patterns

Extracted via regex from ship-track page HTML:

| Carrier | Pattern |
|---------|---------|
| AMZL (Amazon Logistics) | `TBA` followed by 12–15 digits |
| UPS | `1Z[A-Z0-9]{16}` |
| USPS | `9[0-9]{19,21}` |
| FedEx | `[1-8][0-9]{14}` |

## Sync Behavior

- Looks back 60 days or to `amazonLastSync`, whichever gives a shorter window
- Iterates order list pages until all orders in window are processed
- Fetches detail page + up to 3 ship-track pages per order

## Integration Files

| File | Purpose |
|------|---------|
| `src/content/amazon.ts` | Browser extension content script — order list, detail fetch, tracking extraction |
