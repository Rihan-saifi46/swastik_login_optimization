# Swastik Jewels — Dhanteras Flash Sale & ₹1,000 Booking Integration
## CRM Developer Technical Handoff & Backend Architecture Specification

**Target Developer**: Swastik Jewels CRM & Backend Engineering Team  
**Scope Boundary**: Backend, Database, Payments, Webhook Signature Verification, Inventory Locking, and Final Draft Order Checkout Generation.  
*(Shopify storefront frontend UI, modals, buttons, and countdown timers are already implemented in `Login-opt`).*

---

## 1. Architectural Responsibility Matrix

| Component | Responsibility | Current Status |
| :--- | :--- | :---: |
| **"Book Now — ₹1,000" Button** | Shopify Theme (`sections/product.liquid`) | **IMPLEMENTED** |
| **Dhanteras Promotional Banner & Live Countdown** | Shopify Theme (`sections/product.liquid`) | **IMPLEMENTED** |
| **Luxury Booking Modal & Statutory Disclosures** | Shopify Theme (`snippets/dhanteras-booking-modal.liquid`) | **IMPLEMENTED** |
| **Dynamic 30% Discount & Balance Calculation UI** | Shopify Theme (`snippets/dhanteras-booking-modal.liquid`) | **IMPLEMENTED** |
| **Customer Contact Capture (Name, WhatsApp, Email)** | Shopify Theme (`snippets/dhanteras-booking-modal.liquid`) | **IMPLEMENTED** |
| **Out-of-Stock & Reserved State Handling** | Shopify Theme (`sections/product.liquid`) | **IMPLEMENTED** |
| **Order Creation API (`POST /create-booking-order`)** | **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Payment Gateway Integration (Paytm Payment Gateway)** | **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Paytm Callback & Checksum Verification (`POST /paytm-callback`)**| **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Single-Piece Inventory Locking (Shopify Admin API)** | **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Customer Booking Record & Status Lifecycle** | **CRM Database** | **REQUIRED** |
| **4-Hour Flash Sale Validation (Nov 6, 2–6 PM IST)** | **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Shopify Draft Order Creation (₹1,000 Credit + 30% OFF)**| **CRM Backend (`backend-api.swastikjewel.in`)** | **REQUIRED** |
| **Priority WhatsApp & Email Checkout Dispatch** | **CRM Notification Service** | **REQUIRED** |

---

## 2. Required CRM Backend Endpoints

### Endpoint 1: Initiate Booking Order
* **HTTP Method**: `POST`
* **Route**: `/api/dhanteras/create-booking-order`
* **Purpose**: Called by the storefront booking modal when customer clicks **"PAY ₹1,000 TO RESERVE NOW"**. Validates single-piece availability and creates a ₹1,000 Paytm order transaction.

#### Request Payload (sent from frontend):
```json
{
  "product_id": "8472910384",
  "variant_id": "4591028472",
  "sku": "SJ-100",
  "booking_amount": 1000,
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
3. **Paytm Order Initiation**: Call Paytm Initiate Transaction API (`/theia/api/v1/initiateTransaction`) or generate a Paytm Checkout / Payment Link for ₹1,000.00 INR.
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
    "TXN_AMOUNT": "1000.00",
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
4. **Shopify Inventory Locking (Admin REST or GraphQL API)**:
   * Call `POST /admin/api/2024-01/inventory_levels/set.json`:
     Set `available = 0` for this variant at the primary location.
   * Call `PUT /admin/api/2024-01/products/{product_id}.json`:
     Add tag: `dhanteras-reserved:<booking_id>`.
5. **CRM Database Update**:
   * Set status to `CONFIRMED`.
   * Save `paytm_txn_id: paytmParams.TXNID`, `bank_txn_id: paytmParams.BANKTXNID`, `paid_at: ISO8601`, `booking_amount: 1000`.
6. **Customer Notification**:
   * Trigger WhatsApp confirmation message via Wati/Interakt/Gupshup:
     *"Namaste Aditi, your exclusive piece (SKU: SJ-100) is reserved for Dhanteras! Booking ID: SW-DT-2026-894102. Your 30% discount purchase window opens on Nov 6, 2:00 PM - 6:00 PM IST."*
7. **Redirect to Storefront Confirmation**:
   The CRM redirects the browser back to the product page with success parameters:
   ```javascript
   const redirectUrl = storedBooking.redirect_url || 'https://swastikjewel.in';
   res.redirect(`${redirectUrl}?booking_status=success&booking_id=${storedBooking.booking_id}`);
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
3. **Price Calculation**:
   * Example:
     * Product Normal Price: ₹10,000
     * 30% Flash Sale Discount: ₹3,000
     * Paid Booking Deposit Credit: ₹1,000
     * **Total Discount to Apply**: ₹4,000
     * **Final Balance Due**: ₹6,000
4. **Shopify Draft Order Generation (`POST /admin/api/2024-01/draft_orders.json`)**:
   ```json
   {
     "draft_order": {
       "line_items": [
         {
           "variant_id": 4591028472,
           "quantity": 1,
           "applied_discount": {
             "title": "Dhanteras 30% Flash Sale + ₹1,000 Booking Credit",
             "description": "30% Flash discount (-₹3,000) and ₹1,000 reservation deposit credited.",
             "value": 4000,
             "value_type": "fixed_amount",
             "amount": 4000
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
5. **Invoice Dispatch**: The API returns `draft_order.invoice_url`. When the customer clicks or opens this link, they are taken directly into native Shopify Checkout with all shipping, taxes, and payment methods available to pay the remaining ₹6,000 balance.

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

## 3. Database Schema Recommendations

### Table: `dhanteras_bookings`
* `id`: Primary Key (UUID)
* `booking_id`: String (Indexed, Unique, e.g. `SW-DT-2026-894102`)
* `product_id`: BigInt (Shopify Product ID)
* `variant_id`: BigInt (Shopify Variant ID)
* `sku`: String (Jewellery SKU)
* `customer_name`: String
* `customer_phone`: String
* `customer_email`: String
* `booking_amount`: Integer (1000)
* `gateway_order_id`: String
* `gateway_payment_id`: String
* `status`: Enum (`PENDING_PAYMENT`, `CONFIRMED`, `PURCHASED`, `EXPIRED`, `CANCELLED`, `REFUNDED`)
* `created_at`: Timestamp
* `confirmed_at`: Timestamp
* `purchased_at`: Timestamp
* `shopify_order_id`: BigInt (nullable)

---

## 4. Legal & Regulatory Compliance (Statutory Notice)

Under the **Consumer Protection (E-Commerce) Rules, 2020** and applicable Indian contract law:
1. **Transparent Forfeiture**: Because Swastik Jewels withdraws unique 1-of-1 physical pieces from public commerce, retaining the ₹1,000 deposit upon failure to complete purchase within the clearly disclosed 4-hour window is legally permissible, provided the customer affirmatively agreed to the disclosure (which the frontend checkbox enforces).
2. **Mandatory Seller Refund Clause**: If Swastik Jewels cancels the booking due to inventory loss, catalog error, or unfulfillability, the ₹1,000 must be remitted back to the source payment method within 24 to 48 hours.

---

## 5. Summary of Frontend Status

* **Frontend Theme Repository**: `Login-opt`
* **Modal Trigger**: Handled in [`snippets/dhanteras-booking-modal.liquid`](file:///d:/swastik%20shopify%20R/Login-opt/snippets/dhanteras-booking-modal.liquid).
* **Current Fallback State**: The frontend directly calls `https://backend-api.swastikjewel.in/api/dhanteras/create-booking-order`. If the endpoint is unreachable or returns non-200, it displays an honest notice asking the customer to check back shortly or connect via WhatsApp.
* **No fake bookings or mock transactions exist in the storefront code.**
