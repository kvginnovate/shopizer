# 9. Technologies Used in Shopizer — Implementation Status

Inventory of every technology/integration the Shopizer stack references,
vs. what is actually running in our Docker demo. Use this to decide what
else to wire up for the interview.

## Core runtime (WORKING in demo)

| Technology | Role | Status |
|------------|------|--------|
| Java 11 (OpenJDK) | Backend runtime | Working (prebuilt image) |
| Spring Boot 2.5 | Backend framework | Working |
| H2 (embedded) | Database | Working (default profile) |
| Infinispan 9.4 | CMS file storage + cache | Working (default CMS method) |
| Ehcache | Hibernate 2nd-level cache | Working (embedded) |
| Tomcat 9 | Servlet container | Working |
| Nginx | Admin & shop static serving | Working |
| Angular 8 (admin) | Admin UI | Working (prebuilt image) |
| React 16 (shop) | Shop UI | Working (rebuilt image) |
| Redux / react-router | Shop state & routing | Working |
| Docker | Containerization | Working |

## Search (NOT working in demo)

| Technology | Role | Status |
|------------|------|--------|
| OpenSearch 1.x (via `shopizer-search-opensearch-spring-boot-starter`) | Product search index | **Not enabled in demo** — `search.noindex=true` in default profile; OpenSearch container can be run on 9200 but backend config (host list) needs a rebuild + `search.noindex=false` to index. Search page returns no results. |
| | | To enable: run OpenSearch on `localhost:9200`/`9201`, set `search.noindex=false`, rebuild backend. |

## Payments (mocked in demo)

| Technology | Role | Status |
|------------|------|--------|
| Stripe | Card payments | Replaced with MoneyOrder mock (test key only). |
| PayPal Express / REST | PayPal payments | Not configured. |
| Braintree | Card payments | Not configured. |
| BeanStream / Nuvei | Card payments | Not configured. |
| MoneyOrder | Offline payment (no external call) | **Active** (used for demo). |

## Shipping (partially working)

| Technology | Role | Status |
|------------|------|--------|
| Canada Post | Shipping quotes | Module present (`shipping-canadapost-spring-boot-starter`); not configured (no API keys). |
| UPS / custom rules (Drools) | Shipping rules | Default rules present; UPS not configured. |
| Google Maps Distance | Price-by-distance shipping | Key empty (`config.shippingDistancePreProcessor.apiKey`). |

## CMS / File storage (working with default)

| Technology | Role | Status |
|------------|------|--------|
| Infinispan (default) | CMS file store | Working. |
| AWS S3 | CMS file store | Not configured (`config.cms.aws.bucket` empty). |
| GCP Cloud Storage | CMS file store | Not configured (`config.cms.gcp.bucket` empty). |
| HTTPD (external web server) | CMS file store | Not configured. |

## Email (working with default)

| Technology | Role | Status |
|------------|------|--------|
| Spring HtmlEmailSender (SMTP) | Email | Working (default, configured via admin console). |
| AWS SES | Email | Not configured (`config.emailSender=default`). |

## Other integrations

| Technology | Role | Status |
|------------|------|--------|
| Google Maps API | Address autocomplete / map widget | Key empty — widget degrades (console errors). |
| reCAPTCHA | Bot protection on signup | Testing keys configured (works in test mode). |
| MySQL 8 | Alternative database | Container started but backend uses H2. |
| AWS S3 SDK | File uploads | Dependency present, not configured. |

## Summary — what's NOT implemented in the demo

1. **OpenSearch** (product search) — biggest missing piece; `search.noindex=true`.
2. **Real payment providers** (Stripe/PayPal/Braintree) — mocked with MoneyOrder.
3. **Canada Post / UPS shipping** — no API keys.
4. **AWS S3 / GCP storage** — using default Infinispan CMS.
5. **AWS SES email** — using default SMTP.
6. **Google Maps** — empty key (map widget broken but non-fatal).
7. **MySQL** — container running but backend on H2.

For the interview, the demo covers the **core commerce flow** (catalog, cart, checkout, order) plus admin management. The items above are optional integrations that the codebase supports but that require external credentials / services.