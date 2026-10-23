# Swastik Jewels — Dhanteras Flash Sale & ₹999 Booking Integration
## CRM Developer Technical Handoff & Backend Architecture Specification

**Target Developer**: Swastik Jewels CRM & Backend Engineering Team  
**Scope Boundary**: Backend, Database, Payments, Webhook Signature Verification, Inventory Locking, and Final Draft Order Checkout Generation.  
*(Shopify storefront frontend UI, modals, buttons, and countdown timers are already implemented in `Login-opt`).*

> [!IMPORTANT]
> **DEVELOPER INSTRUCTION**: Please read the **NEW CHANGES — ACTION REQUIRED** section below first. Consult the remainder of this document only for supporting technical details, architectural context, and existing API contracts.

---

# NEW CHANGES — ACTION REQUIRED
**Change date: 23 October 2026**

### Purpose of this Update
This section outlines the immediate backend and API deliverables required following the storefront theme update in `Login-opt`. It allows the CRM & backend engineering team to identify active development tasks immediately without auditing the entire historical document.

> [!NOTE]
> The storefront frontend UI (PDP, product cards, and Dhanteras reservation modal) has been updated in Shopify theme `Login-opt`. The backend API contracts, CRM database, and Shopify Admin API checkout discount mechanisms must now be verified and implemented by the CRM team to match this business logic. Do not assume backend changes are complete.

---

### 1. Booking Amount Changed from ₹1,000 to ₹999

The Dhanteras reservation deposit has been changed from **₹1,000 to ₹999** across the storefront.

#### Backend & Payment Gateway Deliverables:
1. **Endpoint 1 (`POST /api/dhanteras/create-booking-order`)**:
   - The storefront now sends `"booking_amount": 999` in the JSON request payload.
   - The backend must accept `999` (validate that `booking_amount === 999` and reject requests claiming other amounts).
2. **Paytm Payment Gateway Initiation (Actual Payment Amount)**:
   - When calling Paytm Initiate Transaction API (`/theia/api/v1/initiateTransaction`) or generating a Paytm Checkout payment link, configure the authoritative transaction amount as:
     `TXN_AMOUNT: "999.00"` (previously `"1000.00"`).
   - Ensure the actual Paytm payment collected is ₹999, not just the frontend display.
3. **CRM Database Persistence**:
   - Save `booking_amount: 999` in the `dhanteras_bookings` table for pending and confirmed records.
4. **Draft Order Checkout Balance Calculation (`POST /api/dhanteras/generate-sale-checkout`)**:
   - The ₹999 paid reservation deposit must be credited against the final balance due **exactly once**:
     $$\text{final\_balance\_due} = \text{sale\_product\_price} - \text{booking\_deposit\_credit} \; (\text{₹999})$$
     $$\text{draft\_order\_applied\_discount} = \text{diamond\_discount} + \text{booking\_deposit\_credit} \; (\text{₹999})$$
5. **Strict Scope Restriction**:
   - Do not change unrelated ₹1,000 amounts outside this Dhanteras reservation flow.

---

### 2. Diamond-Only 30% Discount Calculation

The Shopify storefront theme and the Dhanteras Book Now reservation modal have been updated to require that the 30% flash sale discount applies **ONLY to the diamond component/value**, and **NOT** to the complete jewellery product price.

#### Backend Requirements:
1. **Verify Authoritative Diamond Component Field**:
   - Check whether the CRM backend/API already exposes the authoritative diamond component value for each eligible product.
   - The authoritative Shopify data source is the variant metafield:
     `variant.metafields.custom.diamond_value` (stored in INR rupees).
   - If this field is not yet integrated or exposed in the CRM API/database, define and implement the required API/data-model change to store and serve it.
2. **Apply 30% Discount Strictly to Diamond Component**:
   - The backend/Shopify discount mechanism (e.g. Draft Orders, Shopify Functions, or automatic checkout generation) must apply the 30% discount strictly to the eligible diamond component:
     $$\text{diamond\_discount} = \text{diamond\_component\_value} \times 0.30$$
3. **Calculate Product Sale Price**:
   - The product sale price must be calculated as the original product price minus the diamond discount:
     $$\text{sale\_product\_price} = \text{original\_product\_price} - \text{diamond\_discount}$$
4. **Credit ₹999 Booking Deposit Against Balance**:
   - The ₹999 paid booking deposit must be credited against the final balance due exactly once:
     $$\text{final\_balance\_due} = \text{sale\_product\_price} - \text{booking\_deposit\_credit} \; (\text{₹999})$$
     $$\text{draft\_order\_applied\_discount} = \text{diamond\_discount} + \text{booking\_deposit\_credit} \; (\text{₹999})$$
5. **Ensure Storefront & Checkout Price Consistency**:
   - The reservation modal, product detail page (PDP), product listing cards, and final Shopify checkout draft order invoice must all use the exact same authoritative calculation.
6. **Strict Prohibition**:
   - **Do NOT** calculate the discount as $\text{original\_product\_price} \times 0.30$.
   - **Do NOT** estimate the diamond component as an arbitrary percentage of the total price.
   - **Do NOT** invent diamond values. If a product or variant has no diamond component (`diamond_value <= 0` or missing), the discount is **₹0**, and only the ₹999 advance deposit is credited.

*(For full draft order payload schema and examples, refer to [Section 2 — Endpoint 3: Priority Flash Sale Checkout Generation](#endpoint-3-priority-flash-sale-checkout-generation)).*

---

### 3. Dhanteras Reservation Modal Pricing

The Book Now reservation modal ([`snippets/dhanteras-booking-modal.liquid`](file:///d:/swastik%20shopify%20R/Login-opt/snippets/dhanteras-booking-modal.liquid)) now displays the corrected diamond-only discount calculation and ₹999 reservation amount across all customer-facing rows:
- **Normal Product Price**: Full original variant price (e.g. ₹53,716)
- **Dhanteras 30% Flash Sale**: Sale price after deducting 30% of diamond component (e.g. ₹50,716 for ₹10,000 diamond value)
- **Advance Booking Deposit**: ₹999 credit
- **Final Payable on Nov 6th (2–6 PM)**: Net balance payable during the flash sale (e.g. ₹49,717)

#### CRM Developer Verification:
- **Data Availability**: Confirm that the reservation flow and backend endpoints can retrieve or receive the correct variant diamond component value.
- **Final Balance Accuracy**: Confirm that the final payable balance is calculated correctly after deducting the ₹999 deposit exactly once.
- **End-to-End Consistency**: Ensure the actual order/checkout discount generated by the CRM matches the agreed diamond-only pricing rule and ₹999 deposit.
- **Preserve Existing Flows**: Ensure that existing single-piece inventory locking (`available = 0`), `BOOKED` badge synchronization, customer contact capture, and Paytm payment gateway flows remain completely intact without alterations.

*(For reservation order initiation payload and Paytm flow, refer to [Section 2 — Endpoint 1 & Endpoint 2](#endpoint-1-initiate-booking-order)).*

---

### 4. Existing Diamond Sale Status and Scheduling

Review the existing CRM/backend requirements for the scheduled Diamond Sale and implement any outstanding items already specified in this handoff document:

- **Sale Date**: 6 November 2026.
- **Start Time**: 2:00:00 PM IST (`14:00:00+05:30` / `1793953800` Unix timestamp).
- **End Time**: 6:00:00 PM IST (`18:00:00+05:30` / `1793968200` Unix timestamp).
- **Discount Scope**: 30% on the diamond component only.
- **Schedule & Eligibility Enforcement**:
  - Implement and verify the sale status endpoint (`GET /api/diamond-sale/status` or `/api/dhanteras/sale-status`).
  - Confirm that the backend and Shopify integration enforce the correct schedule and eligibility at checkout so that orders placed before 2:00 PM IST or after 6:00 PM IST are not discounted.
  - Automate the Shopify discount lifecycle (creation, activation at 2:00 PM, deactivation at 6:00 PM) where applicable.
  - **Do not assume previously documented endpoints or features already exist**; verify the actual backend implementation against the production repository.

---

### Developer Action Checklist

- [ ] Verify booking deposit amount update: Accept `booking_amount: 999` in `POST /api/dhanteras/create-booking-order`.
- [ ] Configure Paytm transaction initiation with `TXN_AMOUNT: "999.00"`.
- [ ] Persist `booking_amount: 999` in CRM database (`dhanteras_bookings`).
- [ ] Verify the authoritative diamond component value and its API field (`variant.metafields.custom.diamond_value`).
- [ ] Implement any missing backend/API changes to expose and process diamond component values.
- [ ] Apply diamond-only discount logic + ₹999 deposit credit to the actual Shopify checkout/order mechanism (Draft Orders / Admin API).
- [ ] Validate reservation deposit (₹999) and final payable calculations across test products.
- [ ] Verify sale scheduling (6 Nov 2026, 2:00 PM – 6:00 PM IST), eligibility rules, and hard cut-off enforcement.
- [ ] Test Paytm callback, signature verification, and existing reservation functionality.
- [ ] Report the changed backend files, endpoints, and test results to the tech lead.

---

### Change Log
- **23 October 2026 (Update 2)**: Updated reservation booking amount from ₹1,000 to ₹999 across storefront banner, modal, button labels, API request payload, Paytm transaction initiation amount, and Draft Order balance credit.
- **23 October 2026 (Update 1)**: Added "NEW CHANGES — ACTION REQUIRED" section specifying diamond-only 30% discount calculation rule, reservation modal pricing alignment, and backend verification checklist.
- **Pre-23 October 2026**: Initial Dhanteras reservation integration, Paytm callback, inventory locking, and status sync specification.

---

## 1. Architectural Responsibility Matrix

| Component | Responsibility | Current Status |
| :--- | :--- | :---: |
| **"Book Now — ₹999" Button** | Shopify Theme (`sections/product.liquid`) | **IMPLEMENTED** |
| **Dhanteras Promotional Banner & Live Countdown** | Shopify Theme (`sections/product.liquid`) | **IMPLEMENTED** |
| **Luxury Booking Modal & Statutory Disclosures** | Shopify Theme (`snippets/dhanteras-booking-modal.liquid`) | **IMPLEMENTED** |
| **Dynamic 30% Discount & Balance Calculation UI** | Shopify Theme (`snippets/dhanteras-booking-modal.liquid`) | **IMPLEMENTED** |
| **Customer Contact Capture (Name, WhatsApp, Email)** | Shopify Theme (`snippets/dhanteras-booking-modal.liquid`) | **IMPLEMENTED** |
| **Out-of-Stock & Reserved State Handling** | Shopify Theme (`sections/product.liquid`) | **IMPLEMENTED** |
| **Order Creation API (`POST /create-booking-order`)** | **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Payment Gateway Integration (Paytm Payment Gateway)** | **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Paytm Callback & Checksum Verification (`POST /paytm-callback`)**| **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Single-Piece Inventory Locking (Shopify Admin API)** | **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Shopify Booking Status Sync (BOOKED Badge vs. Out of Stock)** | **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Cancellation & Inventory Restoration Flow** | **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Customer Booking Record & Status Lifecycle** | **CRM Database** | **REQUIRED** |
| **4-Hour Flash Sale Validation (Nov 6, 2–6 PM IST)** | **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Shopify Draft Order Creation (₹999 Credit + 30% OFF)**| **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Priority WhatsApp & Email Checkout Dispatch** | **CRM Notification Service** | **REQUIRED** |

---

## 2. Required CRM Backend Endpoints

### Endpoint 1: Initiate Booking Order
* **HTTP Method**: `POST`
* **Route**: `/api/dhanteras/create-booking-order`
* **Purpose**: Called by the storefront booking modal when customer clicks **"PAY ₹999 TO RESERVE NOW"**. Validates single-piece availability and creates a ₹999 Paytm order transaction.

#### Request Payload (sent from frontend):
```json
{
  "product_id": "8472910384",
  "variant_id": "4591028472",
  "sku": "SJ-100",
  "booking_amount": 999,
  "customer_name": "Aditi Sharma",
  "customer_phone": "+919876543210",
  "customer_email": "aditi@example.com",
  "source": "shopify_storefront",
  "gateway": "paytm",
  "redirect_url": "https://swastikjewel.in/products/exclusive-diamond-necklace"
}
```

#### Backend Requirements:
1. **Inventory Pre-Check**: Call Shopify Admin API (`GET /admin/api/2024-01/inventory_levels.json`) to confirm that the requested `variant_id` currently has `available >= 1`.
2. **Concurrent Lock Prevention**: In the CRM database, ensure no other booking exists with status `CONFIRMED` or `PENDING_PAYMENT` (within a 10-minute hold window) for the same `variant_id`.
3. **Paytm Order Initiation**: Call Paytm Initiate Transaction API (`/theia/api/v1/initiateTransaction`) or generate a Paytm Checkout / Payment Link for ₹999.00 INR.
4. **CRM Database Insertion**: Insert a booking record with status `PENDING_PAYMENT`.

#### Success Response Options (HTTP 200):

**Option A (Recommended — Direct Redirect URL):**
```json
{
  "status": "success",
  "booking_id": "SW-DT-2026-894102",
  "checkout_url": "https://securegw.paytm.in/theia/api/v1/showPaymentPage?mid=YOUR_MID&orderId=SW-DT-2026-894102&txnToken=YOUR_TXN_TOKEN"
}
```
*(The storefront immediately redirects customer to this URL via `window.location.href`).*

**Option B (Paytm Standard Form POST):**
```json
{
  "status": "success",
  "booking_id": "SW-DT-2026-894102",
  "paytm_post_url": "https://securegw.paytm.in/theia/processTransaction",
  "paytm_params": {
    "MID": "YOUR_MID",
    "ORDER_ID": "SW-DT-2026-894102",
    "TXN_AMOUNT": "999.00",
    "WEBSITE": "DEFAULT",
    "CHANNEL_ID": "WEB",
    "INDUSTRY_TYPE_ID": "Retail",
    "CHECKSUMHASH": "e7c65c2b6a22...",
    "CALLBACK_URL": "https://backend-api.swastikjewel.in/api/dhanteras/paytm-callback"
  }
}
```
*(The storefront automatically constructs and posts a hidden form to Paytm).*

#### Error Response Expected by Frontend:
```json
// If item is already booked or sold out (HTTP 409 Conflict):
{
  "status": "error",
  "message": "Sorry, this exclusive single piece has just been reserved by another customer."
}

// If inputs are invalid (HTTP 400 Bad Request):
{
  "status": "error",
  "message": "Invalid customer phone number or email address."
}
```

---

### Endpoint 2: Paytm Callback, Checksum Verification & Inventory Locking
* **HTTP Method**: `POST`
* **Route**: `/api/dhanteras/paytm-callback` (or configured Paytm Callback URL)
* **Purpose**: Receives transaction response from Paytm, cryptographically verifies checksum signature, transitions booking status to `CONFIRMED`, atomically locks the 1-of-1 inventory in Shopify, and redirects customer back to the storefront confirmation screen.

#### Backend Requirements:
1. **Paytm Checksum Verification**:
   ```javascript
   const PaytmChecksum = require('paytmchecksum');
   
   const paytmParams = req.body;
   const paytmChecksum = paytmParams.CHECKSUMHASH;
   delete paytmParams.CHECKSUMHASH;

   const isVerifySignature = PaytmChecksum.verifySignatureByString(
     JSON.stringify(paytmParams), 
     process.env.PAYTM_MERCHANT_KEY, 
     paytmChecksum
   );

   if (!isVerifySignature) {
     return res.status(400).send("Checksum Mismatch");
   }
   ```
2. **Status Check**:
   If `paytmParams.STATUS === 'TXN_SUCCESS'` (or `RESPCODE === '01'`):
3. **Idempotency**: If `booking_id` is already `CONFIRMED`, skip re-executing inventory decrements.
4. **Shopify Inventory Locking & Reservation Identification (Admin REST or GraphQL API)**:
   * Call `POST /admin/api/2024-01/inventory_levels/set.json`:
     Set `available = 0` for this variant at the primary location.
   * Synchronize confirmed booking identification to Shopify so the storefront displays **`BOOKED`** instead of `OUT OF STOCK` across PDP and listing cards (refer to Section 3 for multi-variant vs. 1-of-1 tagging specification).
5. **CRM Database Update**:
   * Set status to `CONFIRMED`.
   * Save `paytm_txn_id: paytmParams.TXNID`, `bank_txn_id: paytmParams.BANKTXNID`, `paid_at: ISO8601`, `booking_amount: 999`.
6. **Customer Notification**:
   * Trigger WhatsApp confirmation message via Wati/Interakt/Gupshup:
     *"Namaste Aditi, your exclusive piece (SKU: SJ-100) is reserved for Dhanteras! Booking ID: SW-DT-2026-894102. Your 30% discount purchase window opens on Nov 6, 2:00 PM - 6:00 PM IST."*
7. **Redirect to Storefront Confirmation**:
   The CRM redirects the browser back to the product page with success parameters:
   ```javascript
   const redirectUrl = storedBooking.redirect_url || 'https://swastikjewel.in';
   res.redirect(`${redirectUrl}?booking_status=success&booking_id=${storedBooking.booking_id}&variant_id=${storedBooking.variant_id}`);
   ```
   *(The storefront automatically catches `booking_status=success` and pops open the Confirmation screen with the Booking ID).*

   If payment failed or cancelled:
   ```javascript
   res.redirect(`${storedBooking.redirect_url}?booking_status=failed&booking_id=${storedBooking.booking_id}`);
   ```
```

---

### Endpoint 3: Priority Flash Sale Checkout Generation
* **HTTP Method**: `POST`
* **Route**: `/api/dhanteras/generate-sale-checkout`
* **Purpose**: Executed during the strictly limited 4-hour flash sale window on **November 6, 2026 (2:00 PM – 6:00 PM IST)** to generate the final discounted checkout invoice.

#### Request Payload:
```json
{
  "booking_id": "SW-DT-2026-894102",
  "customer_phone": "+919876543210"
}
```

#### Backend Requirements:
1. **Strict 4-Hour Time Window Enforcement**:
   ```javascript
   const saleStart = new Date('2026-11-06T14:00:00+05:30').getTime();
   const saleEnd   = new Date('2026-11-06T18:00:00+05:30').getTime();
   const now = Date.now();

   if (now < saleStart) {
     return res.status(403).json({ 
       status: 'error', 
       message: 'The Dhanteras Flash Sale begins on Nov 6 at 2:00 PM IST.' 
     });
   }
   if (now > saleEnd) {
     return res.status(403).json({ 
       status: 'error', 
       message: 'The Dhanteras 4-hour flash sale window has closed. As per reservation terms, unredeemed bookings have expired.' 
     });
   }
   ```
2. **Booking Validation**: Confirm booking exists, status is `CONFIRMED`, and has not already been `PURCHASED` (prevents double redemption).
3. **Price Calculation (Diamond-Only 30% Flash Sale Rule)**:
   * **Authoritative Business Rule**: The 30% discount applies **ONLY** to the diamond component/value, never to the full jewellery product price.
   * **Authoritative Data Source**: The variant diamond component is retrieved from Shopify Admin API Variant Metafields:
     `variant.metafields.custom.diamond_value` (stored in INR rupees).
   * **Formula**:
     $$\text{diamond\_discount} = \text{diamond\_component\_value} \times 0.30$$
     $$\text{sale\_product\_price} = \text{original\_product\_price} - \text{diamond\_discount}$$
     $$\text{final\_balance\_due} = \text{sale\_product\_price} - \text{booking\_deposit\_credit}$$
     $$\text{draft\_order\_applied\_discount} = \text{diamond\_discount} + \text{booking\_deposit\_credit}$$
   * **Missing Data / Non-Diamond Handling**:
     If `diamond_component_value` is 0 or missing, $\text{diamond\_discount} = 0$. The sale product price equals the original product price, and the draft order applied discount is strictly the ₹999 booking deposit credit. Under no circumstances should the CRM apply 30% to the full product price or estimate diamond value.
   * **Illustrative Product Example**:
     * Original Product Price: ₹53,716
     * Diamond Component Value (`metafields.custom.diamond_value`): ₹10,000
     * 30% Diamond Discount: ₹3,000 ($10,000 \times 0.30$)
     * Sale Product Price: ₹50,716 ($53,716 - 3,000$)
     * Paid Booking Deposit Credit: ₹999
     * **Total Discount to Apply to Draft Order**: ₹3,999 ($3,000 + 999$)
     * **Final Balance Due**: ₹49,717 ($50,716 - 999$)
4. **Shopify Draft Order Generation (`POST /admin/api/2024-01/draft_orders.json`)**:
   ```json
   {
     "draft_order": {
       "line_items": [
         {
           "variant_id": 4591028472,
           "quantity": 1,
           "applied_discount": {
             "title": "Dhanteras 30% Diamond Discount + ₹999 Booking Credit",
             "description": "30% off diamond component (-₹3,000) and ₹999 reservation deposit credited.",
             "value": 3999,
             "value_type": "fixed_amount",
             "amount": 3999
           }
         }
       ],
       "customer": {
         "email": "aditi@example.com"
       },
       "tags": "dhanteras-flash-order:SW-DT-2026-894102",
       "use_customer_default_address": true
     }
   }
   ```
5. **Invoice Dispatch**: The API returns `draft_order.invoice_url`. When the customer clicks or opens this link, they are taken directly into native Shopify Checkout with all shipping, taxes, and payment methods available to pay the remaining ₹49,717 balance.

#### Success Response (HTTP 200):
```json
{
  "status": "success",
  "invoice_url": "https://swastikjewel.in/62910482/checkouts/do/c7b28a910.../invoice"
}
```

---

### Webhook Listener: Shopify Order Creation / Paid
* **Route**: `/api/dhanteras/shopify-order-webhook`
* **Trigger**: Shopify `orders/create` or `orders/paid`
* **Logic**:
  1. Inspect order tags or note attributes for `dhanteras-flash-order:<booking_id>`.
  2. If present, update CRM booking status from `CONFIRMED` to `PURCHASED`.
  3. Record Shopify `order_id` in the CRM booking table for complete financial auditing.

---

## 3. Shopify Booking Status Sync — BOOKED After Successful Reservation

### 3.1 Architectural Context & Problem Statement
In Swastik Jewels' 1-of-1 exclusive jewellery model, reserving a product previously resulted in setting Shopify inventory to `0` (`available = 0`). Consequently, the storefront defaulted to displaying `OUT OF STOCK`.

The Shopify frontend (`Login-opt`) has been updated to differentiate between an unbooked out-of-stock item and an active reservation. When a confirmed booking exists:
* The availability badge across the Product Details Page (`sections/product.liquid`), product listing cards (`snippets/product-card.liquid`), and recommendations displays **`BOOKED`** instead of `OUT OF STOCK`.
* Purchase actions ("Add to Cart", "Book Now", "Buy Now") transition into disabled **`BOOKED`** states.
* Products that have not been reserved preserve their normal `IN STOCK` or `MADE TO ORDER` states.

To make the `BOOKED` status universally persistent across all devices, browser sessions, page reloads, and different customers, the CRM backend must synchronize confirmed bookings to Shopify using the existing CRM-to-Shopify integration.

---

### 3.2 Paytm Payment Gateway Confirmation & CRM State Persistence
* **Actual Production Gateway**: **Paytm** (`/api/dhanteras/paytm-callback`).
* **Trigger Event**: Paytm server-to-server callback or customer return with `STATUS === 'TXN_SUCCESS'` (or `RESPCODE === '01'`).
* **Backend Processing Sequence**:
  1. **Signature Verification**: Verify Paytm signature using `PaytmChecksum.verifySignatureByString(JSON.stringify(paytmParams), process.env.PAYTM_MERCHANT_KEY, paytmChecksum)`.
  2. **Idempotent State Transition**:
     - Locate the booking record in table `dhanteras_bookings` using `booking_id` (e.g. `SW-DT-2026-894102`).
     - If status is already `CONFIRMED`, skip re-executing inventory decrements or duplicate tag calls.
     - If status is `PENDING_PAYMENT`, transition status to `CONFIRMED`.
     - Persist transaction metadata: `paytm_txn_id`, `bank_txn_id`, `paid_at: ISO8601`, `booking_amount: 999`.
  3. **Immediate Storefront Redirection**:
     - Include the booked variant ID in the redirect URL:
       ```javascript
       const redirectUrl = storedBooking.redirect_url || 'https://swastikjewel.in';
       res.redirect(`${redirectUrl}?booking_status=success&booking_id=${storedBooking.booking_id}&variant_id=${storedBooking.variant_id}`);
       ```
     - *Client Persistence*: The frontend catches `booking_status=success` and immediately caches `{ [variant_id]: { booking_id, product_id, timestamp } }` in `localStorage['sw_confirmed_bookings']`, eliminating any perceived visual delay while Shopify CDN indexes the product update.

---

### 3.3 Shopify CRM-to-Shopify Synchronization Specification
The CRM backend must sync the reservation to Shopify using the existing Shopify Admin REST or GraphQL API credentials.

#### 1. Inventory Decrement (Withdraw Item from Sale)
* Call `POST /admin/api/2024-01/inventory_levels/set.json`:
  - `location_id`: Primary fulfillment location ID
  - `inventory_item_id`: Target variant's `inventory_item_id`
  - `available`: `0`

#### 2. Booking Identification: Proposal Validation (Tags vs. Metafields)
The CRM integration must apply the reservation at the correct entity level so unrelated products or variants are never incorrectly marked `BOOKED`:

* **Case A: Multi-Variant Products** (e.g. products with options for metal colour, purity, or ring size):
  * **Tag Proposal (Recommended for Simplicity)**:
    - Append tag to product via `PUT /admin/api/2024-01/products/{product_id}.json`:
      `dhanteras-reserved:<variant_id>` or `booked:<variant_id>`
      *(Example: `dhanteras-reserved:4591028472`)*
    - *Storefront Execution*: The Liquid and JS engines verify `tag contains 'dhanteras-reserved:4591028472'`. Only variant `4591028472` displays `BOOKED`; selecting any other variant of that product displays `IN STOCK` or `MADE TO ORDER`.
  * **Metafield Proposal (Alternative via GraphQL Admin API)**:
    - Mutate variant metafield on the target `variant_id`:
      - Namespace: `custom`
      - Key: `reserved` (Type: `boolean`, Value: `true`), OR Key: `booking_status` (Type: `single_line_text_field`, Value: `"confirmed"`)

* **Case B: Single-Variant / 1-of-1 Piece Products** (`product.has_only_default_variant` or single item):
  * **Tag Proposal**:
    - Append tag to product: `dhanteras-reserved:<booking_id>` or `dhanteras-reserved` or `booked`
    - *Example*: `dhanteras-reserved:SW-DT-2026-894102`
  * **Metafield Proposal**:
    - Mutate product-level metafield:
      - Namespace: `custom`
      - Key: `reserved` (Type: `boolean`, Value: `true`), OR Key: `booking_status` (Type: `single_line_text_field`, Value: `"confirmed"`)

> **Note for Backend / CRM Team**: The frontend code in `Login-opt` has already implemented dual support for both tag patterns (`dhanteras-reserved:*`, `booked:*`) and metafield keys (`custom.reserved`, `custom.booked`, `custom.booking_status`). Treat these tag and metafield formats as proposals to validate against your existing Shopify Admin API integration tooling, and select the one that integrates most seamlessly into your stack.

---

### 3.4 Cross-Browser, Multi-Session & Refresh Persistence
* **Server-Rendered Status**: Because the CRM updates the tag or metafield on the Shopify product record, the Liquid engine in `product-card.liquid`, `product.liquid`, and `product-recommendations.liquid` renders `<span class="product-card-ui__pill product-card-ui__pill--booked">BOOKED</span>` and `<div class="pp-stock-status pp-stock-status--booked"><span>BOOKED</span></div>` on the server.
* **Universal Visibility**: Any visitor across any browser, mobile device, or new session will see `BOOKED` rather than `OUT OF STOCK`.

---

### 3.5 Handling Failed, Timed-Out, or Unconfirmed Bookings
* **Failed Bookings**:
  - If Paytm returns `STATUS === 'TXN_FAILURE'`, customer aborts checkout, or transaction fails validation, update CRM record to `FAILED`.
  - **Never mark the product as BOOKED in Shopify for failed or unconfirmed bookings.**
  - The existing availability (`IN STOCK` or `MADE TO ORDER`) must remain unchanged.
* **Unconfirmed Pending Bookings (Hold Expiry)**:
  - If a 10-minute hold window was tracked in the CRM during `PENDING_PAYMENT`, and no successful Paytm callback is received within 10 minutes, transition the CRM status to `EXPIRED`.
  - Do not apply any persistent reservation tag/metafield in Shopify.

---

### 3.6 Cancellation & Inventory Restoration Flow
If a confirmed booking is cancelled (e.g. customer service cancellation, merchant release, or post-flash-sale expiry on Nov 6):

1. **CRM Record Update**: Update status in table `dhanteras_bookings` to `CANCELLED` or `REFUNDED`.
2. **Remove Shopify Booking Identification**:
   * If using tags: Remove `dhanteras-reserved:<variant_id>`, `dhanteras-reserved:<booking_id>`, or `dhanteras-reserved` from the product tags via `PUT /admin/api/2024-01/products/{product_id}.json`.
   * If using metafields: Set `custom.reserved = false` or `custom.booking_status = "cancelled"`.
3. **Restore Shopify Inventory Quantity**:
   * Call `POST /admin/api/2024-01/inventory_levels/adjust.json`:
     - `inventory_item_id`: Target variant's `inventory_item_id`
     - `available_adjustment`: `1` (or set inventory back to `1`)
4. **Storefront Result**:
   * The product badge automatically reverts to `IN STOCK` (or `MADE TO ORDER` if tagged with `Made to Order` and policy `continue`).
   * "Add to Cart", "Book Now", and "Buy Now" buttons are immediately re-enabled.

---

### 3.7 Architecture & Scope Boundaries
* **Payment Gateway**: Paytm is the exclusive production gateway for this integration. Do not implement Razorpay-specific workflows or credentials.
* **Endpoint Architecture**: No new API endpoints are required. All logic attaches to existing routes:
  - `/api/dhanteras/create-booking-order` (Pre-check & Paytm initiation)
  - `/api/dhanteras/paytm-callback` (Checksum verification, DB confirmation, Shopify sync, customer redirect)
  - `/api/dhanteras/generate-sale-checkout` (Flash sale Nov 6 draft order generation)
* **Database Schema**: Reuses existing `dhanteras_bookings` table without structural alterations.

---

## 4. Database Schema Recommendations

### Table: `dhanteras_bookings`
* `id`: Primary Key (UUID)
* `booking_id`: String (Indexed, Unique, e.g. `SW-DT-2026-894102`)
* `product_id`: BigInt (Shopify Product ID)
* `variant_id`: BigInt (Shopify Variant ID)
* `sku`: String (Jewellery SKU)
* `customer_name`: String
* `customer_phone`: String
* `customer_email`: String
* `booking_amount`: Integer (999)
* `gateway_order_id`: String
* `gateway_payment_id`: String
* `status`: Enum (`PENDING_PAYMENT`, `CONFIRMED`, `PURCHASED`, `EXPIRED`, `CANCELLED`, `REFUNDED`)
* `created_at`: Timestamp
* `confirmed_at`: Timestamp
* `purchased_at`: Timestamp
* `shopify_order_id`: BigInt (nullable)

---

## 5. Legal & Regulatory Compliance (Statutory Notice)

Under the **Consumer Protection (E-Commerce) Rules, 2020** and applicable Indian contract law:
1. **Transparent Forfeiture**: Because Swastik Jewels withdraws unique 1-of-1 physical pieces from public commerce, retaining the ₹999 deposit upon failure to complete purchase within the clearly disclosed 4-hour window is legally permissible, provided the customer affirmatively agreed to the disclosure (which the frontend checkbox enforces).
2. **Mandatory Seller Refund Clause**: If Swastik Jewels cancels the booking due to inventory loss, catalog error, or unfulfillability, the ₹999 must be remitted back to the source payment method within 24 to 48 hours.

---

## 6. Summary of Frontend Status

* **Frontend Theme Repository**: `Login-opt`
* **Modal Trigger**: Handled in [`snippets/dhanteras-booking-modal.liquid`](file:///d:/swastik%20shopify%20R/Login-opt/snippets/dhanteras-booking-modal.liquid).
* **Current Fallback State**: The frontend directly calls `https://backend-api.swastikjewel.in/api/dhanteras/create-booking-order`. If the endpoint is unreachable or returns non-200, it displays an honest notice asking the customer to check back shortly or connect via WhatsApp.
* **No fake bookings or mock transactions exist in the storefront code.**
