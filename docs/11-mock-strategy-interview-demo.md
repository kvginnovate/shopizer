# 11. Mock Strategy for Interview Demo

What we mocked, how, and what else can be mocked to show a complete
end-to-end flow without external services/credentials.

## Already mocked (working in demo)

### 1. Payment — MoneyOrder (mock)
- **What:** Replaced Stripe (empty key) with the built-in `MoneyOrderPayment` module,
  which authorizes & captures without calling any external provider.
- **How:**
  - Backend: `POST /api/v1/private/modules/payment?store=DEFAULT` with
    `{"code":"moneyorder","active":true,"defaultSelected":true,"integrationKeys":{"address":"123 Test St"}}`.
  - Shop: `APP_PAYMENT_TYPE=MONEYORDER` in `.env` / `env-config.js`.
  - Shop `Checkout.js`: added a MoneyOrder payment block (terms checkbox + place-order
    button) that calls `onPayment(data, null)` → sends `paymentType: MONEYORDER`,
    `paymentModule: moneyorder`, no `paymentToken`.
  - Backend `OrderFacadeImpl` builds a `Payment` with `PaymentType.MONEYORDER` and
    `MoneyOrderPayment.authorizeAndCapture()` returns a successful `Transaction`
    without any external call.
- **Result:** Order is created in the DB with `paymentType=MONEYORDER`, `orderStatus=ORDERED`.

### 2. Product images — empty array guarded
- **What:** Products have `images: []` in seed data. Product detail page crashed on
  `product.images[0].imageUrl`.
- **How:** Guarded `useState` and render conditionals in `ProductImageGallery.js`
  and `defaultImage()` helpers. Shows placeholder icon instead of crashing.
- **To make it look better:** upload product images via admin or set a placeholder
  image URL in the seed data / `defaultImage()`.

### 3. Google Maps — degrades gracefully
- **What:** `APP_MAP_API_KEY` empty → map widget shows "REQUEST_DENIED" console error.
- **How:** Guarded `Geocode.setApiKey()` in `userAction.js` (only sets if key present).
  The checkout form still works (country/zone are manual selects).
- **To mock:** set a free Google Maps API key, or remove the map widget.

### 4. Shipping quotes — null (no shipping configured)
- **What:** Shipping country API returns zones but no shipping module configured
  → `shippingOptions: null`.
- **How:** `onChangeShipping` calls `shippingQuoteChange('')` when null. The checkout
  sends `shippingQuote: ''` (empty). Backend handles null shipping.
- **Result:** Order created without shipping module. Total = subtotal.

### 5. Zone codes — null in seed data
- **What:** All zones have `code: null`. Backend `zoneService.getByCode()` returns null.
- **How:** `CustomerFacadeImpl` falls back to matching zone by description name
  (e.g., "Quebec" matches the zone named "Quebec"). Also, checkout sends
  `zone: ""` (empty) to skip zone lookup entirely.
- **Result:** Checkout/registration works without zone codes.

### 6. Missing content boxes
- **What:** `headerMessage` and (potentially) `agreement` content boxes don't exist.
- **How:** Created `headerMessage` via admin API. `agreement` fetch is caught (no crash).
- **To mock:** create the `agreement` box via admin API with terms text.

### 7. Search — returns empty
- **What:** OpenSearch not running; `search.noindex=true`.
- **How:** Search endpoint returns `{"productCount":0,"products":[]}` (no crash).
- **To mock:** return all products from the product list endpoint instead of search.

## What else can be mocked

### 8. Product images (easy)
- **Mock:** Set a placeholder image URL in `defaultImage()` helpers and
  `ProductImageGallery` when `images` is empty. Use a placeholder service like
  `https://via.placeholder.com/300x300?text=Product`.
- **Files:** `Cart.js`, `ProductGridSingle.js`, `ProductGridListSingle.js`,
  `ProductImageGallery.js`, `MenuCart.js`.

### 9. Search results (easy)
- **Mock:** In `SearchProduct.js`, if search returns 0, fall back to fetching all
  products from `/api/v1/products?store=DEFAULT&lang=en`.
- **Or:** Disable the search link in the header (hide search icon).

### 10. Shipping quotes (moderate)
- **Mock:** Hardcode a flat shipping rate in `onChangeShipping` instead of calling
  the backend shipping API. Set `selectedOptions` to a dummy ID.
- **Or:** Configure a shipping module in admin (flat rate, $5).

### 11. Email confirmation (moderate)
- **Mock:** The backend sends email via SMTP (default). If SMTP isn't configured,
  the email silently fails but the order is still created.
- **To verify:** check backend logs for email errors (non-fatal).

### 12. Customer registration (moderate)
- **Mock:** The zone fallback (CustomerFacadeImpl) handles named zones. For the demo,
  use guest checkout (no registration needed) — the checkout flow creates an
  anonymous customer automatically.

### 13. ReCAPTCHA (already mocked)
- **What:** Testing keys are hardcoded in `shopizer-core.properties`.
- **Status:** Works in test mode (always passes).

### 14. Admin orders display (workaround)
- **What:** Admin orders page 500s when selecting a store (running image bug).
- **Mock:** Set `localStorage.setItem('merchant','DEFAULT')` before loading the
  admin orders page. Or verify orders via the backend API / Swagger.

## Recommended demo flow (fully mocked, no external services)

1. **Shop** `localhost:8082`:
   - Browse catalogue (products show placeholder images).
   - Add chair to cart → cart page (totals shown, product link works).
   - Checkout → fill billing (Canada, Quebec) → check terms → **Place order**.
   - Order created → redirected to `/order-confirm` with order ID.
2. **Admin** `localhost:8081` (admin@shopizer.com / password):
   - Products list (3 products) — select DEFAULT store.
   - Orders list — verify via API: `GET /api/v1/private/orders?store=DEFAULT`.
3. **Backend** `localhost:8080/swagger-ui.html`:
   - Execute store/product/order endpoints live.
   - Show the MoneyOrder payment module configuration.

## What cannot be mocked easily
- Real payment (Stripe charge) — needs a real test secret key + Stripe account.
- Real shipping (Canada Post/UPS) — needs API keys.
- Real email delivery — needs SMTP server config.
- Product search — needs OpenSearch + backend rebuild.
- Cloud storage (S3/GCP) — needs credentials.