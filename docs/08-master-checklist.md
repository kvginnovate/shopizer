# 8. Master Checklist — Work Done & Remaining

Snapshot of everything performed to make the Shopizer stack demo-ready,
plus the items still open. Use this to drive the next work session.

## A. Verification & Diagnosis (DONE)

- [x] Confirmed Docker daemon running; listed containers (all were `Exited (255)`).
- [x] Started `shopizer-mysql`, `shopizer-backend`, `shopizer-admin`, `shopizer-shop`.
- [x] Polled backend until `/api/v1/store/DEFAULT` returned HTTP 200 (~35s).
- [x] Loaded all three app URLs via BrowserOS neo:
  - Shop `http://localhost:8082` (home rendered).
  - Admin `http://localhost:8081` (login rendered).
  - Swagger `http://localhost:8080/swagger-ui.html` (full API catalog).
- [x] End-to-end shop flow verified: home → category → add to cart → cart → checkout.
- [x] Admin login verified (`admin@shopizer.com` / `password`).
- [x] Admin product list verified (3 products after selecting DEFAULT store).
- [x] Backend order creation verified via MoneyOrder (2 orders in DB).
- [x] Console errors captured (Stripe empty key, Google Maps key missing, country duplication).

## B. Code Fixes (DONE in source, committed locally)

### Backend (`shopizer`)
- [x] `ImagesController.java` — guard missing `static/not-found.png` (no ERROR log).
- [x] `ReadableMinimalProductMapper.java` — set `friendlyUrl` from `getSeUrl()` (fixes `/product/null` cart links).
- [x] `AuthenticateCustomerApi.java` — throw `ServiceRuntimeException` (400) instead of NPE on missing country.
- [x] `CustomerFacadeImpl.java` — zone-by-name fallback when code lookup fails.
- [x] `mvnw` — LF line endings (Windows fix).

### Shop frontend (`shopizer-shop-reactjs`)
- [x] `ProductDetail.js` — fetch by route slug so direct URLs/refresh work.
- [x] `Cart.js` — fallback product link to `sku` when `friendlyUrl` is null.
- [x] `Checkout.js` — MoneyOrder payment block + place-order button; moneyorder payment params.
- [x] `ProductImageGallery.js` — guard against empty `images` array.
- [x] `userAction.js` — dedupe shipping countries by code; guard empty Maps key.
- [x] `env-config.js`, `public/env-config.js`, `.env` — `APP_PAYMENT_TYPE=MONEYORDER`.

### Admin (`shopizer-admin`)
- [x] `products-list.component.ts` — default store filter to `DEFAULT`.
- [x] `order-list.component.ts` — default store filter to `DEFAULT`.
- [x] `environment.prod.ts` — default language `en`.

## C. Runtime / Config (DONE, live)

- [x] Activated MoneyOrder payment module for DEFAULT store (admin API).
- [x] Rebuilt shop Docker image (`shopizer-shop-local`) with all fixes + LF line endings.
- [x] Ran new shop image on port 8082.
- [x] Updated shop container `env-config.js` (`APP_PAYMENT_TYPE=MONEYORDER`, Stripe test key).
- [x] Updated admin container `assets/env.js` (`APP_DEFAULT_LANGUAGE=en`).
- [x] Created missing `headerMessage` content box via admin API.
- [x] Verified admin now renders in English.
- [x] Verified product detail, cart links, checkout button work in rebuilt image.

## D. Documentation (DONE)

- [x] `docs/06-docker-setup.md` (pre-existing).
- [x] `docs/07-replication-and-customization.md` (replication + demo steps).
- [x] `docs/08-master-checklist.md` (this file).

## E. Git / Push (DONE)

- [x] Committed changes in `shopizer`, `shopizer-admin`, `shopizer-shop-reactjs`.
- [x] Forked all three to `kvginnovate/*` and pushed.
- [x] Added `docs/07-replication-and-customization.md` to shopizer fork.

## F. Open / Pending Items (TODO)

### High priority
- [ ] Rebuild backend jar with Java 11 (not 21) so the backend code fixes
      (friendlyUrl, zone fallback, ImagesController, AuthenticateCustomerApi) actually
      deploy. Currently only MoneyOrder config + H2 data are live.
- [ ] Rebuild admin Angular image (`npm install && npm run build` then `docker build`)
      so the store-filter default and English language code changes take effect
      (currently only the env.js runtime tweak is live).
- [ ] Fix admin orders page 500 when selecting a store via the combobox
      (possibly a param format issue; needs investigation).
- [ ] Seed product images so product detail page shows a real image
      (currently a placeholder icon; gallery guarded against empty array).

### Medium priority
- [ ] Make checkout resilient to the first-attempt `ConcurrentModificationException`
      (async search indexing side effect) — retry logic or disable indexing.
- [ ] Set a real Google Maps API key or remove the map widget to eliminate
      console errors.
- [ ] Populate zone codes in the seeded H2 data (or keep the name fallback).
- [ ] Add a proper payment success/confirmation redirect (currently stays on checkout).

### Low priority / nice-to-have
- [ ] Configure a real Stripe test secret on the backend for a true card payment demo.
- [ ] Add sample orders/customers for a richer admin demo.
- [ ] Clean up CRLF line endings in all shell scripts (Windows dev ergonomics).
- [ ] Document the transient 500/retry behavior for the interviewer.

## G. Known Limitations (by design for the demo)

- Payment is mocked via MoneyOrder (no real charge).
- Backend code fixes are not deployed to the running container (prebuilt image).
- Admin requires manual store selection until image is rebuilt.
- No product images in seed data.