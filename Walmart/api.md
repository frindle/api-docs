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

## Sync Behavior

- Syncs back 30 days, or to `walmartLastSync - 48 hours`, whichever is earlier
- Fetches up to 3 detail pages concurrently

## Integration Files

| File | Purpose |
|------|---------|
| `src/content/walmart.ts` | Browser extension content script — order list DOM scrape, `__NEXT_DATA__` detail parse |
