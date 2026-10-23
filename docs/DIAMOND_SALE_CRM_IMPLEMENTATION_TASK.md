# Swastik Jewels — CRM/Backend Developer Implementation Task
## Scheduled 30% Diamond Sale (6 November 2026, 2:00 PM – 6:00 PM IST)
### Automated Server-Authoritative Discount Enforcement & Schedule Management

**Target Developer**: Swastik Jewels CRM & Backend Engineering Team  
**Deliverable Path**: `docs/DIAMOND_SALE_CRM_IMPLEMENTATION_TASK.md`  
**Related Theme Files**: [`snippets/diamond-sale-logic.liquid`](file:///d:/swastik%20shopify%20R/Login-opt/snippets/diamond-sale-logic.liquid), [`sections/product.liquid`](file:///d:/swastik%20shopify%20R/Login-opt/sections/product.liquid), [`sections/cart.liquid`](file:///d:/swastik%20shopify%20R/Login-opt/sections/cart.liquid), [`docs/DIAMOND_SALE_CRM_DEVELOPER_HANDOFF.md`](file:///d:/swastik%20shopify%20R/Login-opt/docs/DIAMOND_SALE_CRM_DEVELOPER_HANDOFF.md)  
**Document Version**: 1.0.0 (Implementation-Ready Task Specification)

> [!IMPORTANT]
> **DEVELOPER INSTRUCTION**: Please read the **NEW CHANGES — ACTION REQUIRED** section below first. Consult the remainder of this task specification only for supporting implementation details, schemas, and GraphQL mutation samples.

---

# NEW CHANGES — ACTION REQUIRED
**Change date: 23 October 2026**

### Purpose of this Update
This section outlines the latest pending implementation work and updated pricing rules following the Shopify storefront updates in `Login-opt`. It allows the CRM & backend engineering team to identify active development tasks immediately without reviewing every historical requirement in this document.

> [!NOTE]
> The storefront frontend UI (PDP, product cards, and Dhanteras reservation modal) has been updated in Shopify theme `Login-opt`. The backend API contracts, CRM database, and Shopify Admin API checkout discount mechanisms must now be verified and implemented by the CRM team to match this business logic. Do not assume backend changes are complete.

---

### 1. Diamond-Only 30% Discount Enforcement

The 30% flash sale discount must apply **STRICTLY to the eligible diamond component/value**, and **NOT** to the complete jewellery product price.

#### Backend Implementation Tasks:
1. **Verify Authoritative Diamond Component Field**:
   - Confirm the CRM exposes the authoritative diamond component value for each eligible SKU.
   - Authoritative Shopify data source: `variant.metafields.custom.diamond_value` (INR rupees).
   - If missing from backend data models, add the required field to `diamond_sale_configs` / product metadata tables.
2. **Implement Calculation Endpoint (`POST /api/diamond-sale/calculate-order`)**:
   $$\text{diamond\_discount} = \text{diamond\_component\_value} \times 0.30$$
   $$\text{sale\_product\_price} = \text{original\_product\_price} - \text{diamond\_discount}$$
3. **Strict Calculation Prohibitions**:
   - **Do NOT** calculate the discount as $\text{original\_product\_price} \times 0.30$.
   - **Do NOT** estimate the diamond component as an arbitrary percentage of the total price.
   - **Do NOT** invent diamond values. If a product or variant has no diamond component (`diamond_value <= 0` or missing), the discount is **₹0**.

---

### 2. Pre-Reservation Deposit Credit (Updated from ₹1,000 to ₹999)

For customers with Dhanteras advance reservations checking out during the 4-hour sale window:
1. **Booking Amount Update**:
   - The Dhanteras reservation amount has been changed from ₹1,000 to **₹999**.
   - Verify and update the authoritative backend booking amount, payment request, server-side validation, booking records, and API contract where applicable (refer to [`DHANTERAS_CRM_DEVELOPER_HANDOFF.md`](file:///d:/swastik%20shopify%20R/Login-opt/docs/DHANTERAS_CRM_DEVELOPER_HANDOFF.md)).
   - Ensure the actual Paytm payment amount is ₹999, not just the frontend display.
   - Do not change unrelated ₹1,000 amounts outside this Dhanteras reservation flow.
2. **Draft Order Balance Due Formula (`POST /admin/api/2024-01/draft_orders.json`)**:
   - Deduct the ₹999 deposit credit **exactly once** from the sale price:
     $$\text{final\_balance\_due} = \text{sale\_product\_price} - 999$$
     $$\text{draft\_order\_applied\_discount} = \text{diamond\_discount} + 999$$
   - Example (Product ₹53,716 with ₹10,000 diamond value):
     - Diamond discount (30%): ₹3,000
     - Sale product price: ₹50,716
     - Paid reservation credit: ₹999
     - Total Draft Order applied discount: ₹3,999 ($3,000 + 999$)
     - Final balance due at checkout: **₹49,717** ($50,716 - 999$).

---

### 3. Reservation Modal and Final Payable Integration

- The Dhanteras booking modal ([`snippets/dhanteras-booking-modal.liquid`](file:///d:/swastik%20shopify%20R/Login-opt/snippets/dhanteras-booking-modal.liquid)) uses the diamond-only discount and ₹999 advance deposit.
- The CRM developer must verify the final payable amount and ensure the actual checkout/order amount matches the frontend display.
- Preserve existing single-piece inventory locking (`available = 0`), `BOOKED` badge synchronization, customer login flow, and Paytm payment gateway functionality.

---

### 4. Scheduled Sale Implementation & Verification (6 Nov 2026, 2:00 PM – 6:00 PM IST)

- **Sale Date**: Friday, 6 November 2026.
- **Sale Window**: 2:00:00 PM IST (`14:00:00+05:30` / `2026-11-06T08:30:00Z` UTC) to 6:00:00 PM IST (`18:00:00+05:30` / `2026-11-06T12:30:00Z` UTC).
- **Backend Endpoints Requiring Implementation / Verification**:
  - `GET /api/diamond-sale/status`: Status endpoint returning live server state in IST.
  - `POST /api/diamond-sale/calculate-order`: Server calculation enforcing 30% diamond-only discount.
  - Programmatic Shopify Admin GraphQL discount lifecycle (`discountAutomaticBasicCreate`).
  - Hard cutoff at 6:00:00 PM IST (reject stale carts and pre-2 PM attempts).

---

### Developer Action Checklist

- [ ] Verify the existing backend implementation and API contracts against production codebase.
- [ ] Verify the authoritative diamond component value field (`variant.metafields.custom.diamond_value`).
- [ ] Add the required backend/API field if diamond component value is missing.
- [ ] Update discount calculation in `POST /api/diamond-sale/calculate-order` to 30% of diamond value only.
- [ ] Update the authoritative booking amount to ₹999 for Dhanteras reservations.
- [ ] Verify actual Paytm payment amount is ₹999 and final payable balance deducts ₹999 exactly once.
- [ ] Implement/verify `GET /api/diamond-sale/status` endpoint and schedule enforcement (6 Nov, 2–6 PM IST).
- [ ] Test sale start/end timing, stale cart cutoff (6:00:00 PM IST), and product eligibility rules.
- [ ] Report changed backend files, endpoints, and test results.

---

### Change Log
- **23 October 2026**: Added "NEW CHANGES — ACTION REQUIRED" section specifying diamond-only 30% discount calculation, Dhanteras reservation deposit update to ₹999, Draft Order balance formulas, and backend verification checklist.
- **Pre-23 October 2026**: Initial implementation task specification.

---

## 1. Task Summary & Scope

### 1.1 Objective
Prepare and deploy backend/CRM infrastructure to manage and enforce the upcoming **30% Diamond Flash Sale** automatically, without requiring store staff to manually log into Shopify Admin to create, schedule, enable, or disable discounts for recurring events.

### 1.2 Event Parameters
- **Event Name**: 30% Diamond Flash Sale
- **Sale Date**: Friday, 6 November 2026
- **Sale Start**: 2:00 PM IST (`2026-11-06T14:00:00+05:30` / `2026-11-06T08:30:00Z` UTC)
- **Sale End**: 6:00 PM IST (`2026-11-06T18:00:00+05:30` / `2026-11-06T12:30:00Z` UTC)
- **Duration**: Exactly 4 hours (14,400 seconds)
- **Discount Rate**: 30% applied strictly to the **diamond component/value** only (not to gold/metal, making charges, or GST)
- **Timezone**: `Asia/Kolkata` (IST, UTC+05:30)
- **Payment Gateway**: Existing **Paytm Payment Gateway** integration must be preserved without modification. **No Razorpay** or alternative gateways may be introduced.

### 1.3 Boundary of Responsibilities: Frontend (Theme) vs. Backend (CRM)

```
+-----------------------------------------------------------------------------------+
| STOREFRONT THEME (`Login-opt`) - ALREADY IMPLEMENTED                              |
| - Server-rendered 30% promotional reference price & struck original price         |
| - "30% OFF ON DIAMONDS" badge on eligible products                                |
| - Pre-sale warning modal with mandatory unchecked acknowledgement checkbox        |
| - Buy Now & Cart Checkout interception before 2:00 PM IST                         |
| - Client-side Asia/Kolkata clock sync via HTTP HEAD origin Date header            |
| - Periodic status query to `GET /api/diamond-sale/status`                         |
+-----------------------------------------------------------------------------------+
                                     |
                                     | REST / JSON
                                     v
+-----------------------------------------------------------------------------------+
| CRM / BACKEND (`backend-api.swastikjewel.in`) - REQUIRED IMPLEMENTATION           |
| - Central database configuration for schedule, percentage, and product eligibility|
| - Status endpoint: `GET /api/diamond-sale/status`                                 |
| - Server-authoritative order calculation: `POST /api/diamond-sale/calculate-order`|
| - Programmatic Shopify Admin API discount management (`write_discounts`)          |
| - 1:30 PM IST pre-flight automated health check & alerts                          |
| - Hard cutoff at 6:00:00 PM IST for stale carts & new checkout attempts           |
| - Paytm signature verification (`PaytmChecksum`) & webhook reconciliation        |
+-----------------------------------------------------------------------------------+
                                     |
                                     | Admin GraphQL
                                     v
+-----------------------------------------------------------------------------------+
| SHOPIFY CORE CHECKOUT & PAYTM PAYMENT GATEWAY                                     |
| - Native checkout discount evaluation (enforced server-side)                      |
| - Paytm hosted payment page / form POST checkout                                  |
| - Final payment capture & order generation                                        |
+-----------------------------------------------------------------------------------+
```

---

## 2. Confirmed Business Rules

1. **Pre-Sale Price Enforcement (Before 2:00 PM IST)**:
   - While the storefront displays a promotional reference price to create anticipation, any checkout or order completed prior to 2:00 PM IST **must charge the full regular original price**.
   - The theme intercepts checkout with an unchecked acknowledgement checkbox. The backend/checkout must never authorize a discounted order prior to 2:00 PM IST.
2. **Active Window Enforcement (2:00 PM – 6:00 PM IST)**:
   - Eligible diamond products must automatically receive the real 30% discount at checkout.
   - The customer **must not be required to enter any coupon code**.
   - The final amount payable at Paytm checkout must match the server-calculated eligible price.
3. **Hard Post-Sale Cutoff (After 6:00 PM IST)**:
   - At and after 6:00:00 PM IST, no new checkouts or orders may receive the discount.
   - **Stale Cart Rule**: A checkout session opened before 6:00 PM but finalized at or after 6:00:01 PM IST must be rejected or recalculated at the regular price.
4. **CRM-Managed Lifecycle (Zero Recurring Manual Shopify Admin Action)**:
   - Once the one-time integration and API permissions are configured, recurring sales must be scheduled, activated, paused, or modified exclusively from the CRM dashboard/backend. Store staff must not need to manually configure Shopify Admin discounts.
5. **Anti-Tampering & Server Authority**:
   - The frontend theme includes development helpers (`DiamondSale.setTestPhase(...)` and `?diamond_sale_test_phase=...`). The CRM backend and Shopify checkout must **strictly ignore client-side parameters** and enforce discounts based exclusively on authoritative server clocks and trusted product data.
6. **Payment Gateway Continuity**:
   - Reuses the existing Paytm payment gateway. No Razorpay libraries, keys, or callback endpoints may be introduced.
7. **Coupon Compatibility & Stacking**:
   - By default, the 30% Diamond Sale discount cannot be combined with standard coupon codes (e.g. `WELCOME10`) unless explicitly configured in the CRM.

---

## 3. Backend Discovery & Environment Audit Checklist

The CRM developer must perform this audit on the backend server (`backend-api.swastikjewel.in`) before writing code:

- [ ] **Discovery 1: Runtime Framework & Database**
  - Identify backend framework (Node.js/Express, Python/Django/FastAPI, Go, PHP/Laravel).
  - Identify database system (PostgreSQL, MySQL, MongoDB).
  - Confirm migration tool (e.g. Prisma, TypeORM, Knex, Alembic) to create configuration tables.
- [ ] **Discovery 2: Existing Shopify Custom App Integration**
  - Check whether the CRM already connects to Shopify via a Custom App.
  - Verify if the current Shopify Access Token (`shpat_...`) has the following scopes:
    - `write_discounts`, `read_discounts` *(Mandatory for programmatic discount management)*.
    - `write_products`, `read_products` *(Required for reading product tags, collections, variant prices)*.
    - `write_draft_orders`, `read_draft_orders` *(Required if supporting custom reservation orders)*.
    - `read_orders` *(Required for webhook order audit)*.
  - If scopes are missing, update the Custom App scopes in **Shopify Admin -> Settings -> Apps and sales channels -> Develop apps**.
- [ ] **Discovery 3: Existing Paytm Payment Integration**
  - Locate existing Paytm transaction initiation and callback handler (e.g. in `dhanteras` service).
  - Verify that `PAYTM_MID`, `PAYTM_MERCHANT_KEY`, and `PAYTM_WEBSITE_NAME` are securely loaded from environment variables.
  - Verify that `paytmchecksum` library is available for signature verification.
- [ ] **Discovery 4: Background Task Scheduler**
  - Identify active scheduler/worker framework (BullMQ, Celery, Agenda, Cron, AWS EventBridge).
  - Confirm capacity to run the 1:30 PM pre-flight readiness check and the 6:00 PM cutoff audit.

---

## 4. Proposed Architecture & CRM Data Model

### 4.1 CRM Database Schema

#### Table / Document: `diamond_sale_configs`
Stores the authoritative configuration for the sale event.

```sql
CREATE TABLE IF NOT EXISTS diamond_sale_configs (
    id VARCHAR(64) PRIMARY KEY,                  -- e.g. "dia_sale_20261106"
    title VARCHAR(255) NOT NULL,                 -- "30% Diamond Flash Sale"
    discount_percent NUMERIC(5,2) NOT NULL DEFAULT 30.00,
    start_time_iso TIMESTAMP WITH TIME ZONE NOT NULL, -- 2026-11-06 14:00:00+05:30
    end_time_iso TIMESTAMP WITH TIME ZONE NOT NULL,   -- 2026-11-06 18:00:00+05:30
    timezone VARCHAR(50) NOT NULL DEFAULT 'Asia/Kolkata',
    status VARCHAR(30) NOT NULL DEFAULT 'SCHEDULED',  -- SCHEDULED, ACTIVE, PAUSED, COMPLETED, CANCELLED
    eligible_tags TEXT[] DEFAULT ARRAY['diamond', 'diamonds', 'diamond-sale', 'natural-diamond', 'lab-grown'],
    eligible_collection_handles TEXT[] DEFAULT ARRAY['natural-diamonds', 'lab-grown-diamonds'],
    allow_coupon_stacking BOOLEAN NOT NULL DEFAULT FALSE,
    shopify_discount_node_id VARCHAR(120),       -- "gid://shopify/DiscountAutomaticNode/..."
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_by VARCHAR(100)
);
```

#### Table / Document: `diamond_sale_audit_logs`
Tracks every lifecycle event and administrative action for regulatory and financial audit.

```sql
CREATE TABLE IF NOT EXISTS diamond_sale_audit_logs (
    id SERIAL PRIMARY KEY,
    sale_config_id VARCHAR(64) REFERENCES diamond_sale_configs(id),
    action VARCHAR(50) NOT NULL,                 -- 'CREATED', 'SCHEDULE_UPDATED', 'ACTIVATED', 'PAUSED', 'EXPIRED', 'CHECKOUT_GENERATED'
    performed_by VARCHAR(100) NOT NULL,         -- Admin User ID or 'SYSTEM_SCHEDULER'
    details JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### 4.2 Conflict Prevention Rule
When creating or updating a sale configuration, the CRM backend must validate that no other sale record has status `ACTIVE` or has overlapping start/end timestamps targeting the same product categories:
```javascript
// Validation logic in CRM service
const conflicting = await db.diamond_sale_configs.findOne({
  status: { $in: ['SCHEDULED', 'ACTIVE'] },
  id: { $ne: currentSaleId },
  $or: [
    { start_time_iso: { $lt: newEndIso }, end_time_iso: { $gt: newStartIso } }
  ]
});
if (conflicting) {
  throw new Error(`Conflict: Sale '${conflicting.title}' is already scheduled during this timeframe.`);
}
```

---

## 5. Shopify Discount Integration: Technical Enforcement

### 5.1 Why External API Calls Cannot Directly Mutate Shopify Checkout
Shopify's core checkout pipeline enforces total calculations on its own infrastructure. External systems cannot directly alter live line-item prices on `/checkout` via arbitrary HTTP POST requests. 

To enforce discounts at checkout **without recurring manual Shopify Admin discount configuration**, the CRM backend must use a supported Shopify integration pattern.

---

### 5.2 Primary Mechanism: Programmatic Shopify Admin GraphQL Automatic Discount Lifecycle (Recommended)

#### How It Operates:
1. The CRM acts as the programmatic administrator using its Custom App API token (`write_discounts`).
2. Prior to the sale (or triggered upon CRM schedule creation), the CRM calls Shopify's Admin GraphQL API to create a native Automatic Discount with strict start and end timestamps.
3. Because the discount rule is native to Shopify, customers shopping through regular web browsers receive the 30% discount at checkout **automatically without typing a discount code**.
4. **Authoritative Expiry**: At 6:00:00 PM IST (`12:30:00 UTC`), Shopify's core checkout engine automatically invalidates the discount rule for all subsequent order finalizations.
5. **Real-Time CRM Control**: The CRM dashboard can pause, resume, or cancel the sale at any time by calling `discountAutomaticDeactivate` or `discountAutomaticActivate`.

#### GraphQL Mutation: `discountAutomaticBasicCreate`
```graphql
mutation CreateDiamondSaleDiscount($automaticBasicDiscount: DiscountAutomaticBasicInput!) {
  discountAutomaticBasicCreate(automaticBasicDiscount: $automaticBasicDiscount) {
    automaticDiscountNode {
      id
      automaticDiscount {
        ... on DiscountAutomaticBasic {
          title
          status
          startsAt
          endsAt
          customerGets {
            value {
              ... on DiscountPercentage {
                percentage
              }
            }
            items {
              ... on DiscountCollections {
                collections(first: 5) {
                  nodes {
                    id
                    handle
                  }
                }
              }
            }
          }
        }
      }
    }
    userErrors {
      field
      code
      message
    }
  }
}
```

#### Mutation Variables (computed by CRM from database):
```json
{
  "automaticBasicDiscount": {
    "title": "30% Diamond Flash Sale [CRM Managed]",
    "startsAt": "2026-11-06T08:30:00Z",
    "endsAt": "2026-11-06T12:30:00Z",
    "customerGets": {
      "value": {
        "percentage": 0.30
      },
      "items": {
        "collections": {
          "add": [
            "gid://shopify/Collection/462910482934",
            "gid://shopify/Collection/462910515702"
          ]
        }
      }
    },
    "combinesWith": {
      "orderDiscounts": false,
      "productDiscounts": false,
      "shippingDiscounts": true
    }
  }
}
```

#### GraphQL Mutation: `discountAutomaticDeactivate` (Emergency Pause / Cancel)
```graphql
mutation DeactivateDiamondDiscount($id: ID!) {
  discountAutomaticDeactivate(id: $id) {
    automaticDiscountNode {
      id
      automaticDiscount {
        ... on DiscountAutomaticBasic {
          status
        }
      }
    }
    userErrors {
      field
      message
    }
  }
}
```

---

### 5.3 Alternative Mechanism: Shopify Functions (`type: product_discount`)
For stores utilizing Shopify Functions:
- A custom Function extension is deployed once to Shopify via Shopify CLI.
- The Function reads dynamic scheduling and eligibility rules from `shop.metafields.swastik_crm.diamond_sale_config`.
- The CRM updates this metafield via `mutation metafieldsSet` using Shopify Admin API.
- During cart/checkout computation, Shopify executes the WebAssembly Function in under 5ms, applying 30% discount if `now >= start && now < end`.

---

## 6. Status Endpoint Contract: `GET /api/diamond-sale/status`

The storefront theme in [`snippets/diamond-sale-logic.liquid`](file:///d:/swastik%20shopify%20R/Login-opt/snippets/diamond-sale-logic.liquid) periodically calls this endpoint to sync clock offsets and verify discount availability.

### 6.1 Endpoint Specification
- **Path**: `/api/diamond-sale/status`
- **Method**: `GET`
- **Authentication**: None (Public storefront-facing read endpoint)
- **CORS Headers**:
  - `Access-Control-Allow-Origin`: `https://swastikjewel.in` (and local development origins `http://127.0.0.1:9292`, `http://localhost:9292`)
  - `Access-Control-Allow-Methods`: `GET, OPTIONS`
  - `Access-Control-Allow-Headers`: `Accept, Content-Type`
- **Cache-Control**: `no-cache, no-store, must-revalidate` (Must return real-time atomic time)

### 6.2 Success Response (Pre-Sale Phase — Prior to 2:00 PM IST)
```json
{
  "status": "success",
  "data": {
    "sale_id": "dia_sale_20261106",
    "event_name": "30% Diamond Flash Sale",
    "current_phase": "pre-sale",
    "server_time_iso": "2026-10-20T14:06:00+05:30",
    "server_time_unix": 1792485360000,
    "timezone": "Asia/Kolkata",
    "sale_window": {
      "start_iso": "2026-11-06T14:00:00+05:30",
      "end_iso": "2026-11-06T18:00:00+05:30",
      "start_unix": 1793953800000,
      "end_unix": 1793968200000,
      "duration_hours": 4
    },
    "discount_percent": 30,
    "checkout_discount_active": false,
    "message": "Upcoming 30% Diamond Sale starts Nov 6 at 2:00 PM IST."
  }
}
```

### 6.3 Success Response (Active Flash Sale Phase — 2:00 PM to 6:00 PM IST)
```json
{
  "status": "success",
  "data": {
    "sale_id": "dia_sale_20261106",
    "event_name": "30% Diamond Flash Sale",
    "current_phase": "active",
    "server_time_iso": "2026-11-06T14:15:00+05:30",
    "server_time_unix": 1793954700000,
    "timezone": "Asia/Kolkata",
    "sale_window": {
      "start_iso": "2026-11-06T14:00:00+05:30",
      "end_iso": "2026-11-06T18:00:00+05:30",
      "start_unix": 1793953800000,
      "end_unix": 1793968200000
    },
    "discount_percent": 30,
    "checkout_discount_active": true,
    "shopify_discount_node_id": "gid://shopify/DiscountAutomaticNode/1209384729",
    "message": "30% Diamond Flash Sale is currently active."
  }
}
```

### 6.4 Success Response (Post-Sale Phase — After 6:00 PM IST)
```json
{
  "status": "success",
  "data": {
    "sale_id": "dia_sale_20261106",
    "event_name": "30% Diamond Flash Sale",
    "current_phase": "post-sale",
    "server_time_iso": "2026-11-06T18:05:00+05:30",
    "server_time_unix": 1793968500000,
    "timezone": "Asia/Kolkata",
    "discount_percent": 30,
    "checkout_discount_active": false,
    "message": "The 30% Diamond Flash Sale has concluded. Normal pricing is active."
  }
}
```

### 6.5 Error Response (HTTP 500 Internal Server Error)
```json
{
  "status": "error",
  "code": "SERVER_ERROR",
  "message": "Unable to determine sale status at this moment.",
  "server_time_unix": 1792485360000
}
```
*(When receiving an error, the storefront theme gracefully falls back to its server-rendered Liquid timestamps).*

---

## 7. Order & Checkout Enforcement Rules

### 7.1 Pre-Sale Window (< 2:00 PM IST)
- Storefront displays 30% off promotional reference price, struck-through original price, and badge.
- If a customer clicks Buy Now or Checkout, the pre-sale modal explains that purchasing now incurs regular price, requiring an unchecked acknowledgement checkbox.
- The order/checkout total must be **₹91,969.00** (regular price for a ₹91,969 product).

### 7.2 Active Window (2:00 PM – 6:00 PM IST)
- **Authoritative Business Rule**: The 30% discount applies **ONLY** to the diamond component/value of the jewellery, never to the total product price.
  - $\text{Original Product Price} = \text{gold/metal value} + \text{diamond value} + \text{stone value} + \text{making charges} + \text{GST}$
  - $\text{Diamond Discount} = \text{eligible diamond value} \times 30\%$
  - $\text{Sale Product Price} = \text{original product price} - \text{diamond discount}$
- **Authoritative Data Source**:
  - The variant diamond value is stored in `variant.metafields.custom.diamond_value` (in INR rupees).
  - If a product or variant has no `diamond_value` or $\le 0$, the discount is ₹0 (no full product price discounting is permitted).
- **Illustrative Example**:
  - Original product price: ₹75,286
  - Diamond component: ₹20,000
  - Diamond discount: ₹6,000 ($20,000 \times 0.30$)
  - Correct sale product price: ₹69,286 (NOT ₹52,700.20)
- **Store Example (Harmony Diamond Band)**:
  - Original price: ₹43,579 (Gold: ₹20,124, Diamond: ₹19,950, Making: ₹2,236, GST: ₹1,269)
  - Diamond component: ₹19,950
  - Diamond discount: ₹5,985 ($19,950 \times 0.30$)
  - Correct sale price: ₹37,594
- **Checkout Enforcement Note for Backend Developer**:
  - Standard Shopify Automatic Basic Discounts apply a percentage to the entire line price. Because our sale is **diamond component only**, the backend integration must enforce the discount via:
    1. **Shopify Function (`product_discount`)**: Custom function that reads `variant.metafields.custom.diamond_value` and applies an exact fixed amount equal to `0.30 * diamond_value`.
    2. **CRM Draft Orders / Cart Transform**: Creates the checkout with explicit line item discount amounts equal to the diamond discount.
    3. **Fixed Amount Automatic Discounts**: Pre-computed variant-specific discounts scheduled via the Admin API.
- Ineligible items in the same cart (e.g. plain gold coins) remain at full price:
  $$\text{Cart Total} = \text{DiamondItem}_{\text{discounted}} + \text{IneligibleItem}_{\text{full\_price}}$$
- Checkout proceeds through the existing Paytm Payment Gateway.

### 7.3 Hard Cutoff & Stale Cart Rule (>= 6:00:00 PM IST)
- At 6:00:00 PM IST, the discount is deactivated on Shopify.
- **Stale Cart Scenario**: A user opens `/checkout` at 5:58 PM IST showing ₹64,378.30. They hesitate and submit their Paytm payment at 6:02 PM IST.
  - **Shopify Automatic Discount Behavior**: When Paytm redirects or finalizes with Shopify Checkout, Shopify re-validates discounts. Because `endsAt: 12:30:00Z` has passed, the line item recalculates at regular price (₹91,969.00).
  - **Paytm Payment Validation**: The CRM callback `/paytm-callback` checks the transaction creation timestamp. If the transaction was initiated after 6:00 PM with an unauthorized discount, the transaction must be rejected.

---

## 8. Scheduling & Reliability Plan

### 8.1 Automated Pre-Flight Check (6 Nov 2026 at 1:30 PM IST — 30 Minutes Before Launch)
The CRM background scheduler must run an automated health check at 1:30 PM IST:
1. Verify database connectivity to `diamond_sale_configs`.
2. Test Shopify Admin API authentication (`GET /admin/api/2024-01/shop.json`).
3. Verify that the Automatic Discount node on Shopify exists and is in `SCHEDULED` state with start time `08:30:00Z`.
4. Test connectivity to Paytm Payment Gateway endpoints.
5. If any check fails, dispatch an urgent SMS / WhatsApp alert to the engineering team (`+91...`).

### 8.2 Safe Failure & Outage Degradation
- **If CRM API is Unreachable**: The storefront theme falls back to Liquid server-rendered metadata. Normal regular prices are preserved to prevent revenue loss.
- **If Shopify API Fails during Activation**: The checkout defaults safely to regular pricing. Customers are protected by the pre-sale acknowledgement notice, preventing unverified discount expectations.
- **Emergency Manual Override**: Provide an authenticated CRM endpoint `POST /api/diamond-sale/emergency-activate` allowing the DevOps team to trigger discount activation with exponential backoff retries.

---

## 9. Implementation Steps in Priority Order

### Phase 1: Environment & Discovery (Immediate)
1. Verify Shopify Custom App API access token and confirm `write_discounts` scope.
2. Locate existing Paytm payment callback logic and verify `paytmchecksum` library.

### Phase 2: CRM Persistence & Status API (Days 1–2)
3. Execute database migration for `diamond_sale_configs` and `diamond_sale_audit_logs`.
4. Insert the official sale record for 6 Nov 2026, 2:00 PM – 6:00 PM IST.
5. Implement and deploy `GET /api/diamond-sale/status` with proper CORS headers.

### Phase 3: Programmatic Shopify Discount Client (Days 2–3)
6. Write service class `ShopifyDiscountService` implementing GraphQL mutations `discountAutomaticBasicCreate` and `discountAutomaticDeactivate`.
7. Write CRM admin endpoints/UI to view, edit, pause, and resume the sale.

### Phase 4: Verification & Webhooks (Days 3–4)
8. Verify Paytm callback handles transaction amounts matching the discounted total without errors.
9. Implement Shopify `orders/paid` webhook listener to tag orders with `diamond-sale-2026`.

### Phase 5: Pre-Flight Scheduler & Cutoff (Day 5)
10. Schedule the automated 1:30 PM IST pre-flight check worker.
11. Perform end-to-end dry-run in staging environment.

---

## 10. Comprehensive Test & Acceptance Checklist

| # | Scenario | Action / Input | Expected Result | Verified By |
| :---: | :--- | :--- | :--- | :---: |
| **1** | **Pre-Sale Price Check** | Visit diamond product page before 2:00 PM IST on Nov 6. | Promotional reference price displayed; struck original price; badge visible; warning notice visible. | QA / Theme |
| **2** | **Pre-Sale Buy Now** | Click "BUY NOW" before 2:00 PM IST. | Pre-sale modal opens; checkbox is unchecked; continue button is disabled. Checking box enables button. | QA / Theme |
| **3** | **Pre-Sale Checkout Charge** | Complete purchase before 2:00 PM IST. | **Customer is charged the full regular original price.** No discount applied. | CRM / Paytm |
| **4** | **Sale Activation** | Clock reaches 2:00:00 PM IST. | Theme status transitions to `FLASH SALE LIVE`. Real 30% discount automatically applies at checkout. | Theme / Shopify |
| **5** | **Active Checkout Charge** | Purchase diamond piece between 2:00 PM and 6:00 PM IST. | 30% discount deducted from order total. Paytm charges exact discounted amount. | CRM / Paytm |
| **6** | **Mixed Cart Handling** | Cart with 1 Diamond Ring (₹50,000) + 1 Plain Gold Coin (₹20,000). | Diamond Ring discounted to ₹35,000. Gold coin remains ₹20,000. Total charged: ₹55,000. | Shopify / CRM |
| **7** | **Variant Change in Sale** | Switch variant from 14K to 18K Gold during active sale. | 30% discount recalculates dynamically against selected variant price. | Theme / Shopify |
| **8** | **Coupon Code Stacking** | Customer attempts coupon code on top of 30% diamond discount. | Coupon stacking rejected unless explicitly enabled in CRM config. | Shopify / CRM |
| **9** | **Sale Termination** | Clock reaches 6:00:00 PM IST. | Storefront restores normal prices automatically. Badge and notice hidden. | Theme / Shopify |
| **10** | **Stale Cart Cutoff** | Cart created at 5:58 PM IST, payment submitted at 6:02 PM IST. | Checkout rejects discount; recalculates at regular price before charging. | Shopify / CRM |
| **11** | **Anti-Tampering** | User sets `window.DiamondSale.setTestPhase('active')` in browser before 2 PM. | Client DOM updates preview, but **checkout and Paytm still charge full regular price**. | CRM / Shopify |
| **12** | **Paytm Integrity** | Test full payment cycle via Paytm PG. | Checksum verified via `PaytmChecksum.verifySignatureByString`. Order marked confirmed. No Razorpay. | CRM / Paytm |
| **13** | **API Outage Resilience** | Temporarily take CRM status endpoint offline. | Storefront falls back gracefully to Liquid metadata; checkout charges safely at regular price. | QA / Theme |

---

## 11. Deployment & Rollback Plan

### 11.1 Deployment Sequence
1. Deploy database schema migrations to production CRM database.
2. Deploy backend service with `GET /api/diamond-sale/status` and `ShopifyDiscountService`.
3. In CRM dashboard, create the sale record. Verify that the programmatic discount is registered on Shopify with start time `2026-11-06T08:30:00Z` and status `SCHEDULED`.
4. Verify status endpoint response via `curl -s https://backend-api.swastikjewel.in/api/diamond-sale/status`.

### 11.2 Rollback / Kill-Switch Procedure
If unexpected pricing or API issues occur during the active sale window:
1. **Instant Pause via CRM Dashboard**: Click "EMERGENCY PAUSE".
2. **Backend Execution**: The CRM calls `discountAutomaticDeactivate(id: $shopifyDiscountId)`.
3. **Storefront Update**: The status endpoint immediately returns `checkout_discount_active: false`. The theme resets display within 10 seconds.
4. **Manual Shopify Admin Fallback (Emergency Only)**: If CRM server is completely offline, store admin can log into Shopify Admin -> Discounts -> Click "30% Diamond Flash Sale" -> Click "Deactivate".

---

## 12. Open Questions & Items Requiring CRM Developer Verification

| # | Item Requiring Verification | Verification Method | Action if Missing |
| :---: | :--- | :--- | :--- |
| **1** | Does the CRM's Custom App already have `write_discounts`? | Check Shopify Admin -> Apps -> Develop apps -> Configuration -> Admin API access scopes. | Toggle `write_discounts` and re-save app. |
| **2** | Which collections contain all eligible diamond pieces? | Inspect Shopify Admin -> Products -> Collections for collection GIDs of `natural-diamonds` and `lab-grown-diamonds`. | Add collection GIDs to CRM configuration variables. |
| **3** | Is the background task queue running in production? | Check BullMQ/Celery monitoring dashboard on CRM server. | Ensure worker process is active to run pre-flight check on Nov 6 at 1:30 PM IST. |
| **4** | What is the production callback URL configured in Paytm Dashboard? | Check Paytm Merchant Dashboard -> Developer Settings -> Callback URLs. | Confirm it matches the CRM callback endpoint `/paytm-callback`. |

---

*This document is implementation-ready. The CRM engineering team can begin Phase 1 immediately using the GraphQL mutations, data schemas, and status contracts specified above.*
