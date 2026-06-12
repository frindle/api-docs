# Costco API

Costco's order history is served via a **GraphQL** endpoint authenticated with a Microsoft MSAL OAuth2 token. Scraping is done via a browser extension content script that runs in the MAIN world to intercept auth tokens.

**GraphQL endpoint:** `https://ecom-api.costco.com/ebusiness/order/v1/orders/graphql`

## Authentication

Three-tier fallback for obtaining the Bearer token:

1. **Intercepted token** — captured from the MAIN world via XHR/fetch interception before Costco's own JS fires it
2. **`GET /gettoken`** — a Costco-internal endpoint that returns a fresh token if the user is logged in
3. **MSAL auth code exchange** — POST to `https://signin.costco.com/e0714dd4-784d-46d6-a278-3e29553483eb/b2c_1a_sso_wcs_signup_signin_209/oauth2/v2.0/token` with `grant_type=authorization_code` (triggered by Costco's own login flow)

> **Note:** Refresh token grants (`grant_type=refresh_token`) using `B2C_1A_SSO_WCS_signup_signin_209` fail with `AADB2C90088: Expected Value: B2C_1A_SSO_WCS_signup_signin_184`. The auth code exchange works fine. Use `/gettoken` instead of MSAL refresh for background token renewal.

**Client ID (MSAL):** `a3a5186b-7c89-4b4c-93a8-dd604e930757`

**clientId for API calls** — extracted from:
- URL path: `/app/{uuid}/`
- Or JWT payload of the Bearer token

**Warehouse number** — extracted from cookies or localStorage keys matching `store`, `warehouse`, or `wh`.

## GraphQL Query

### `getOnlineOrders`

```graphql
query getOnlineOrders($startDate: String!, $endDate: String!, $pageNumber: Int, $pageSize: Int, $warehouseNumber: String!) {
  getOnlineOrders(startDate: $startDate, endDate: $endDate, pageNumber: $pageNumber, pageSize: $pageSize, warehouseNumber: $warehouseNumber) {
    pageNumber
    pageSize
    totalNumberOfRecords
    bcOrders {
      orderHeaderId
      orderPlacedDate: orderedDate
      orderNumber: sourceOrderNumber
      orderTotal
      warehouseNumber
      status
      emailAddress
      orderCancelAllowed
      orderPaymentFailed
      orderReturnAllowed
      orderLineItems {
        orderLineItemId
        itemId
        itemNumber
        itemDescription
        lineNumber
        status
        deliveryDate
        shippingType
        shippingTimeFrame
        carrierItemCategory
        carrierContactPhone
        isFSAEligible
        isShipToWarehouse
        shipment {
          trackingNumber
          carrierName
        }
      }
    }
  }
}
```

**Response shape:** `data.getOnlineOrders` is an **array** — the pagination result is always `[0]`. Each element has `pageNumber`, `pageSize`, `totalNumberOfRecords`, and `bcOrders[]`.

**BcOrder fields:**

| Field | Type | Description |
|-------|------|-------------|
| `orderHeaderId` | string | Internal order ID (used in sourceUrl) |
| `orderPlacedDate` (alias: `orderedDate`) | string (ISO) | Order date |
| `orderNumber` (alias: `sourceOrderNumber`) | string | Display order number |
| `orderTotal` | number | Order total in USD |
| `warehouseNumber` | number | Warehouse that processed the order |
| `status` | string | `"Shipped"`, `"Delivered"`, etc. |
| `emailAddress` | string | Account email |
| `orderLineItems` | array | Line items with tracking |

**Tracking filtering:** Skip items where `carrierItemCategory` (lowercased) is `"electronic delivery service"`, `"email delivery"`, or `"email"` — these are digital goods with no shipment.

### `receiptsWithCounts`

Returns in-warehouse and gas station purchase receipts.

**Query variables:** unknown — needs capture. Response suggests date range and warehouse filtering may be supported.

```graphql
query receiptsWithCounts(...) {
  receiptsWithCounts {
    inWarehouse
    gasStation
    carWash
    gasAndCarWash
    receipts {
      warehouseName
      receiptType
      documentType
      transactionDateTime
      transactionBarcode
      transactionType
      total
      totalItemCount
      itemArray { itemNumber }
      tenderArray { tenderTypeCode tenderDescription amountTender }
      couponArray { upcnumberCoupon }
    }
  }
}
```

The query has two calling conventions:

**List shape** — pass date range:
```graphql
query receiptsWithCounts($startDate: String!, $endDate: String!, $documentType: String!, $documentSubType: String!) {
  receiptsWithCounts(startDate: $startDate, endDate: $endDate, documentType: $documentType, documentSubType: $documentSubType) { ... }
}
```
Variables: `documentType: "WarehouseReceiptDetail"`, `documentSubType: ""` (empty string confirmed working). Returns lightweight list — `itemArray` has only `{ itemNumber }`.

**Detail shape** — pass barcode:
```graphql
query receiptsWithCounts($barcode: String!, $documentType: String!) {
  receiptsWithCounts(barcode: $barcode, documentType: $documentType) { ... }
}
```
Variables: `documentType: "WarehouseReceiptDetail"`. Returns single full receipt with all item fields, warehouse address, member number, etc.

**Receipt fields:**

| Field | Type | Description |
|-------|------|-------------|
| `warehouseName` | string | e.g. `"SPRING VALLEY"`, `"SW HENDERSON"` |
| `warehouseShortName` | string | Same as `warehouseName` |
| `warehouseNumber` | number | Numeric warehouse ID (e.g. `1709`) |
| `companyNumber` | number | Always `1` |
| `receiptType` | string | `"In-Warehouse"`, `"Gas Station"` |
| `documentType` | string | `"WarehouseReceiptDetail"` |
| `transactionDateTime` | string (ISO) | Local datetime of purchase |
| `transactionDate` | string (YYYY-MM-DD) | Date only |
| `transactionBarcode` | string | **Receipt/order number** printed on the physical receipt. Encodes `companyNumber + warehouseNumber + registerNumber + transactionNumber + YYMMDDHHSS`. Use as `orderNumber` when creating tracker orders for in-store purchases. |
| `transactionType` | string | `"Sales"` |
| `registerNumber` | number | Register that processed the sale |
| `transactionNumber` | number | Transaction number on that register |
| `operatorNumber` | number | Cashier/operator ID |
| `total` | number | Total charged |
| `subTotal` | number | Pre-tax subtotal |
| `taxes` | number | Tax amount |
| `instantSavings` | number | Coupon/instant savings total |
| `totalItemCount` | number | Number of items |
| `membershipNumber` | string | Member's Costco membership number |
| `warehouseAddress1/2` | string | Street address |
| `warehouseCity/State/Country/PostalCode` | string | Location |
| `itemArray` | array | Line items (see below) |
| `tenderArray` | array | Payment method(s) used |
| `couponArray` | array | Coupons applied (`upcnumberCoupon`) |

**Item fields (`itemArray`):**

| Field | Type | Description |
|-------|------|-------------|
| `itemNumber` | string | Costco item number |
| `itemDescription01` | string | Short description (e.g. `"DOORDASH GC"`) |
| `itemDescription02` | string | Extended description (e.g. `"2-$50 GIFT CARDS"`) |
| `itemDepartmentNumber` | number | Department code |
| `unit` | number | Quantity purchased |
| `amount` | number | Line total |
| `itemUnitPriceAmount` | number | Unit price |
| `taxFlag` | string | `"N"` = not taxed |
| `itemIdentifier` | string \| null | `"E"` = regular item |

**Tender type codes observed:** `"061"` = VISA, `"067"` = WALLET - VISA

---

## Products GraphQL

**Endpoint:** `POST https://ecom-api.costco.com/ebusiness/product/v1/products/graphql`

Enriches item numbers with catalog images and descriptions. Used by the Costco site to display product thumbnails alongside order/receipt line items.

```graphql
query products($clientId: String!, $itemNumbers: [String], $locale: [String], $warehouseNumber: String!) {
  products(clientId: $clientId, itemNumbers: $itemNumbers, locale: $locale, warehouseNumber: $warehouseNumber) {
    catalogData {
      itemNumber
      catEntryId
      published
      fieldData { imageName }
      description { shortDescription }
      additionalFieldData { fsa chdIndicator }
      parentData { fieldData { imageName } }
    }
  }
}
```

Variables: `clientId` = same UUID used for orders API, `locale: ["en-US"]`, `warehouseNumber` = user's warehouse.

`fieldData.imageName` is a Cloudflare Images URL (e.g. `https://bfasset.costco-static.com/...`). `published: false` is normal for older/discontinued items — the image and description are still returned.

---

## Membership Profile

**Endpoint:** `GET https://ecom-api.costco.com/ebusiness/customer/v1/loyalty/membershipprofile/status`

```ts
{
  membershipNumber: number;       // e.g. 111901217349
  memberFirstName: string;
  memberLastName: string;
  verifiedMembership: boolean;
  membershipStatusDescription: "Active";
  membershipType: "Goldstar";
  membershipTier: "Executive";
  expirationDate: string;         // YYYY-MM-DD
  memberSince: string;            // YYYY-MM-DD
  cardStatusDescription: "Active";
}
```

---

## Integration Files

| File | Purpose |
|------|---------|
| `src/content/costco.ts` | Browser extension content script — auth, online order sync, warehouse receipt sync |
| `app/api/costco/receipts/route.ts` | POST (import receipts from extension), GET (list unlinked receipts with order candidates) |
| `app/api/costco/receipts/[barcode]/link/route.ts` | POST (link receipt to order + generate PDF attachment), DELETE (unlink) |
| `lib/costcoReceipt.ts` | Receipt data types + PDF generation via pdfkit |
