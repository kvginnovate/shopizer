# 10. Deep Analysis — Remaining Work

Comprehensive scan of the codebase (backend, shop, admin) identifying
everything still to be done beyond the demo. Organized by priority.

## A. Backend (shopizer) — Functional gaps

### A1. Search (OpenSearch/Elasticsearch) — NOT working
- `search.noindex=true` in default profile → no indexing → search returns 0.
- Backend uses Elasticsearch 7.5.2 REST client; hosts configured as `0.0.0.0:9200/9201`
  (0.0.0.0 is a bind address, not a connect address).
- `SearchDelegateImpl` throws "Cannot search in index [product_en_default] index might not exist".
- **Fix:** run Elasticsearch/OpenSearch, set `search.noindex=false`, fix host config,
  rebuild backend jar. Also fix the `Index 0 out of bounds for length 0` error
  (hosts list parsing).

### A2. Backend code fixes not deployed (prebuilt image)
- `CustomerFacadeImpl` zone-by-name fallback (registration/checkout with named zones).
- `ReadableMinimalProductMapper` friendlyUrl (cart links).
- `ImagesController` missing-image guard.
- `AuthenticateCustomerApi` NPE→400.
- **Fix:** rebuild backend jar with Java 11 and rebuild Docker image.

### A3. Registration flow broken on running backend
- `LoginRegister.onRegister` sends `billing.stateProvince` but not `zone`.
- Backend `CustomerFacadeImpl` calls `zoneService.getByCode(address.getZone())` →
  null (zone codes are null in seed data) → "Unsupported zone code" → registration fails.
- **Fix:** deploy A2 (zone fallback) OR send `zone` from frontend OR populate zone codes.

### A4. LazyInitializationException in async search indexing
- `SearchServiceImpl.index` accesses `Product.categories` outside a session
  (`LazyInitializationException`). Triggered after order creation.
- **Fix:** fetch categories eagerly or open a session in the async method.

### A5. ConcurrentModificationException in Hibernate JDBC registry
- Occurs during order commit (first attempt). Likely a race between the async
  indexing task and the main transaction.
- **Fix:** synchronize or disable async indexing; or retry on the client.

### A6. Zone codes null in seed data
- All zones have `code: null`. Worked around via name fallback (A2).
- **Fix:** populate zone codes in the H2 seed data or the zone lookup.

### A7. Product images missing in seed data
- Products have `images: []`. Product detail page shows placeholder icon.
- **Fix:** seed product images or add via admin ProductImageApi.

### A8. MySQL profile not used
- Backend runs H2 (default profile). MySQL container started but unused.
- **Fix:** switch to `mysql` profile + point datasource at MySQL (for production).

### A9. Payment providers not configured
- Stripe/PayPal/Braintree/Nuvei/BeanStream modules present but inactive.
- **Fix:** configure credentials via admin for real payments (or keep MoneyOrder mock).

### A10. Shipping providers not configured
- Canada Post, UPS, price-by-distance modules present but no API keys.
- **Fix:** configure API keys via admin.

### A11. AWS S3 / GCP / SES not configured
- Optional CMS storage and email. Default (Infinispan + SMTP) works.
- **Fix:** set bucket/credentials env vars if cloud storage/email needed.

### A12. Google Maps key empty
- `config.googleMapsKey` and `APP_MAP_API_KEY` empty → map widget broken.
- **Fix:** set a real Maps API key.

## B. Shop frontend (shopizer-shop-reactjs) — Functional gaps

### B1. Search page returns no results
- Wired to `/api/v1/search` POST but backend search not indexed (A1).
- **Fix:** enable search (A1).

### B2. Admin orders page 500 (fixed in source, not deployed)
- `onSelectStore` used `e.value` on a string event. Fixed to `e`.
- **Fix:** rebuild admin image (blocked by Node 24 toolchain).

### B3. Cart `defaultImage` returns null when no images
- Cart item image shows broken if no images. Guarded in code but returns null.
- **Fix:** provide a placeholder image URL.

### B4. Checkout agreement content box fetch 404
- `/content/boxes/agreement` returns 404 (caught, non-fatal). Created `headerMessage` but not `agreement`.
- **Fix:** create the `agreement` content box via admin API.

### B5. Phone field validation
- `phone` is required with 10-digit minLength. Guest checkout may not need phone.
- **Fix:** make phone optional or auto-fill.

### B6. Shipping address fields required even when not shipping
- `shipFirstName`, etc. are required but only rendered when `isShipping` is true.
- React-hook-form may still validate hidden fields. Needs verification.

### B7. No order confirmation redirect
- After successful checkout, the app stays on the checkout page.
- **Fix:** redirect to `/order-confirm` or show a success modal.

## C. Admin (shopizer-admin) — Functional gaps

### C1. Orders page store filter bug (fixed in source)
- `onSelectStore` used `e.value` on a string. Fixed. Not deployed.

### C2. Products/orders default store filter (fixed in source)
- `loadParams` now defaults to `DEFAULT`. Not deployed.

### C3. Admin default language (fixed via env.js + source)
- Source changed to `en`; env.js set to `en`. Working in running container.

### C4. Admin image rebuild blocked
- Angular 8 build fails on Node 24 (`fibers` native module, `nodejs` not found).
- **Fix:** use Node 12/16 (nvm) or a Docker-based build.

## D. Security observations

### D1. Hardcoded test credentials in repo
- `admin@shopizer.com` / `password` documented and used.
- reCAPTCHA testing keys hardcoded in `shopizer-core.properties`.
- Stripe test publishable key in `env-config.js`.
- **Risk:** Low for demo, but should use env vars for production.

### D2. CORS wide open
- Backend returns `Access-Control-Allow-Origin: *` with credentials.
- **Risk:** Medium. Should restrict to known origins in production.

### D3. No rate limiting on login
- `/api/v1/private/login` has no brute-force protection.
- **Risk:** Medium for production.

### D4. SQL injection surface
- Some native queries use string concatenation (needs audit).
- **Fix:** audit all `@Query` annotations.

### D5. Secrets in git
- `.env` files with keys are committed (test keys only, but pattern is risky).
- **Fix:** gitignore `.env` and use env vars.

## E. Quality / maintainability

### E1. Empty catch blocks
- Many `catch (error) {}` in frontend (silent failures).
- **Fix:** log errors or show user-facing messages.

### E2. `console.log` in production code
- `ProductDetail.js` has `console.log(response)`.
- **Fix:** remove debug logging.

### E3. `TODO` / `FIXME` comments
- 40+ TODOs in backend (mostly in optional modules: AWS S3, GCP, SES, BeanStream, PayPal).
- **Fix:** address or remove stale TODOs.

### E4. Deprecated Infinispan / Ehcache versions
- Infinispan 9.4 and Ehcache 2.x are old. Consider upgrading.

### E5. Java 11 target with Java 21 runtime
- Backend targets Java 11 but runtime is 21. Works but not ideal.

## F. Priority order for next work

1. **Rebuild backend jar** (Java 11) → deploy code fixes + enable search.
2. **Rebuild admin image** (Node 12/16) → deploy store-filter + language fixes.
3. **Seed product images** → product detail looks complete.
4. **Create `agreement` content box** → terms display properly.
5. **Fix registration flow** → zone handling.
6. **Fix async search indexing** → `LazyInitializationException`.
7. **Enable OpenSearch** → product search works.
8. **Configure real payment provider** (optional for production).
9. **Security hardening** (CORS, rate limiting, secrets) for production.