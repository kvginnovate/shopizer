# 7. Replication, Customization & Interview Demo Notes

Companion to `06-docker-setup.md`. Covers the changes made to get a fully
working end-to-end flow (browse → cart → checkout → order) on a fresh machine,
including the mock payment setup used for the demo.

## 0. Prerequisites

- Docker Desktop running (`docker ps` succeeds).
- Free ports `8080` (backend), `8081` (admin), `8082` (shop).
- ~2 GB disk for images.

## 1. Start the base stack (official images)

```bash
docker pull shopizerecomm/shopizer:latest
docker pull shopizerecomm/shopizer-admin:latest
docker pull shopizerecomm/shopizer-shop-reactjs:latest

docker run -d --name shopizer-backend -p 8080:8080 shopizerecomm/shopizer:latest
docker run -d --name shopizer-admin \
  -e "APP_BASE_URL=http://localhost:8080/api" \
  -p 8081:80 shopizerecomm/shopizer-admin:latest
docker run -d --name shopizer-shop \
  -e "APP_MERCHANT=DEFAULT" \
  -e "APP_BASE_URL=http://localhost:8080" \
  -p 8082:80 shopizerecomm/shopizer-shop-reactjs:latest
```

Wait for backend readiness:

```bash
curl http://localhost:8080/api/v1/store/DEFAULT   # expect JSON with "code":"DEFAULT"
```

## 2. Mock payment setup (MoneyOrder, no external credentials)

The official shop image uses Stripe with an empty key, which blocks checkout.
For a demo/interview, switch to the built-in **MoneyOrder** payment module,
which authorizes & captures without calling an external provider.

### 2.1 Activate MoneyOrder on the backend (admin API)

```bash
TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/private/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin@shopizer.com","password":"password"}' \
  | python3 -c 'import sys,json;print(json.load(sys.stdin).get("token",""))')

curl -X POST "http://localhost:8080/api/v1/private/modules/payment?store=DEFAULT" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"code":"moneyorder","active":true,"defaultSelected":true,"integrationKeys":{"address":"123 Test St"}}'
```

Verify: `GET /api/v1/private/modules/payment/moneyorder?store=DEFAULT` → `active: true`.

### 2.2 Configure the shop frontend

In `shopizer-shop-reactjs`:

- `.env` and `public/env-config.js` and `env-config.js`:
  - `APP_PAYMENT_TYPE=MONEYORDER`
  - (Stripe key no longer required.)

### 2.3 Rebuild the shop image

The shop Dockerfile uses `node:13.12.0-alpine`. On Windows the shell scripts
have CRLF line endings, which break the container's `env.sh`. Convert first:

```bash
sed -i 's/\r$//' env.sh .env
docker build -t shopizer-shop-local .
docker rm -f shopizer-shop
docker run -d --name shopizer-shop -p 8082:80 shopizer-shop-local
```

## 3. Code changes (already applied in the repo)

### Backend (`shopizer`)

| File | Change |
|------|--------|
| `sm-shop/.../controller/ImagesController.java` | Guard missing `static/not-found.png` (no ERROR log). |
| `sm-shop/.../mapper/catalog/ReadableMinimalProductMapper.java` | Set `friendlyUrl` from `getSeUrl()` (fixes cart links → `/product/null`). |
| `sm-shop/.../api/v1/customer/AuthenticateCustomerApi.java` | Throw `ServiceRuntimeException` (400) instead of NPE on missing country. |
| `sm-shop/.../customer/facade/CustomerFacadeImpl.java` | Fall back to zone-by-name when code lookup fails (fixes "Unsupported zone code"). |
| `mvnw` | LF line endings (Windows fix). |

### Shop frontend (`shopizer-shop-reactjs`)

| File | Change |
|------|--------|
| `src/pages/product-details/ProductDetail.js` | Fetch by route slug so direct URLs/refresh work. |
| `src/pages/other/Cart.js` | Fallback product link to `sku` when `friendlyUrl` is null. |
| `src/pages/other/Checkout.js` | MoneyOrder payment block + place-order button; moneyorder payment params. |
| `src/components/product/ProductImageGallery.js` | Guard against empty `images` array. |
| `src/redux/actions/userAction.js` | Dedupe shipping countries by code; guard empty Maps key. |
| `env-config.js`, `public/env-config.js`, `.env` | `APP_PAYMENT_TYPE=MONEYORDER`. |

### Admin (`shopizer-admin`)

| File | Change |
|------|--------|
| `src/app/pages/catalogue/products/products-list/...ts` | Default store filter to `DEFAULT`. |
| `src/app/pages/orders/order-list/...ts` | Default store filter to `DEFAULT`. |
| `src/environments/environment.prod.ts` | Default language `en`. |

Admin image is rebuilt from `dist/` (Angular build); on a fresh machine run
`npm install && npm run build` in `shopizer-admin` first, then `docker build`.

## 4. Runtime data fixes (apply via admin API after boot)

- Create the missing `headerMessage` content box:
  `POST /api/v1/private/content/box?store=DEFAULT&lang=en` with
  `{"code":"headerMessage","contentType":"BOX","visible":true,"descriptions":[{"language":"en","name":"headerMessage","description":"<p>Welcome to Shopizer</p>"}]}`.
- MoneyOrder activation (see 2.1).

## 5. Demo walkthrough

1. Shop `http://localhost:8082` → Furniture → add chair → cart → checkout.
2. Fill billing (country Canada, state QC), check "I agree", click **Place your order**.
3. Order created (id returned); verify in admin Orders (select DEFAULT store) or via API.
4. Admin `http://localhost:8081` (admin@shopizer.com / password): Products (3), Orders (N).
5. Backend Swagger `http://localhost:8080/swagger-ui.html`.

## 6. Known caveats

- First checkout attempt may hit a transient Hibernate `ConcurrentModificationException`
  (async search indexing). Retry succeeds.
- Admin order/product lists require selecting the DEFAULT store (or rebuild with the code fix).
- Zone codes in the seeded H2 data are null; the backend falls back to name match.
- Google Maps key is empty (map widget degrades gracefully).