# Costco API

Costco's order history is served via a **GraphQL** endpoint authenticated with a Microsoft MSAL OAuth2 token. Scraping is done via a browser extension content script that runs in the MAIN world to intercept auth tokens.

**GraphQL endpoint:** `https://ecom-api.costco.com/ebusiness/order/v1/orders/graphql`

## Authentication

Three-tier fallback for obtaining the Bearer token:

1. **Intercepted token** — captured from the MAIN world via XHR/fetch interception before Costco's own JS fires it
2. **`GET /gettoken`** — a Costco-internal endpoint that returns a fresh token if the user is logged in
3. **MSAL refresh token grant** — POST to `https://signin.costco.com/.../{tenantId}/oauth2/v2.0/token` with:
   - `grant_type=refresh_token`
   - `client_id=a3a5186b-7c89-4b4c-93a8-dd604e930757`
   - `refresh_token=<token from localStorage>`

**Client ID (MSAL):** `a3a5186b-7c89-4b4c-93a8-dd604e930757`

**clientId for API calls** — extracted from:
- URL path: `/app/{uuid}/`
- Or JWT payload of the Bearer token

**Warehouse number** — extracted from cookies or localStorage keys matching `store`, `warehouse`, or `wh`.

## GraphQL Query

### `getOnlineOrders`

```graphql
query getOnlineOrders(
  $startDate: String
  $endDate: String
  $pageNumber: Int
  $pageSize: Int
  $warehouseNumber: String
) {
  bcOrders(
    startDate: $startDate
    endDate: $endDate
    pageNumber: $pageNumber
    pageSize: $pageSize
    warehouseNumber: $warehouseNumber
  ) {
    orderHeaderId
    orderPlacedDate
    sourceOrderNumber
    orderTotal
    status
    orderLineItems {
      itemDescription
      status
      carrierItemCategory
      shipment {
        trackingNumber
        carrierName
      }
    }
  }
}
```

**BcOrder fields:**

| Field | Type | Description |
|-------|------|-------------|
| `orderHeaderId` | string | Internal order ID |
| `orderPlacedDate` | string (ISO) | Order date |
| `sourceOrderNumber` | string | Display order number |
| `orderTotal` | number | Order total in USD |
| `status` | string | Order status |
| `orderLineItems` | array | Line items with tracking |

**Tracking filtering:** Skip items where `carrierItemCategory` (lowercased) is `"electronic delivery service"`, `"email delivery"`, or `"email"` — these are digital goods with no shipment.

## Integration Files

| File | Purpose |
|------|---------|
| `src/content/costco.ts` | Browser extension content script — auth, GraphQL query, order parsing |
