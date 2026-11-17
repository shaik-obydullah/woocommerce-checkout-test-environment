# WooCommerce Checkout Test Environment

Local WordPress + WooCommerce sandbox for reproducing and debugging
**checkout page "fails to load / submit"** errors before delivering to a client.

## Access

| What | URL | Login |
|------|-----|-------|
| Storefront (Shop) | http://localhost:8085/shop/ | — |
| Cart | http://localhost:8085/cart/ | — |
| Checkout (block-based) | http://localhost:8085/checkout/ | — |
| Admin | http://localhost:8085/wp-admin/ | `admin` / `admin123` |

## What's installed

- WordPress **7.1** (updated over the `wordpress:latest` base)
- WooCommerce **11.1.0** (activated, block-based checkout)
- Theme: Twenty Twenty-Five (active)
- Test products: `Test Product A` (£25, SKU `TEST-A`), `Test Product B` (£40, SKU `TEST-B`)
- Shipping: "UK" zone with flat rate £5.00
- Payment: Cheque + Cash on Delivery enabled (no real gateway needed)
- Store defaults: London (GB), currency GBP, taxes enabled

## Containers / services

- `woocheckout_db` — MySQL 8.0 (volume `db_data`)
- `woocheckout_wp` — WordPress web server, **port 8085**
- `wpcli` — one-shot WP-CLI runner (`docker compose run --rm ...`)

`WP_DEBUG` / `WP_DEBUG_LOG` / `WP_DEBUG_DISPLAY` are enabled so PHP errors are
written to `wp-content/debug.log` (visible on the host at `./wp-content/debug.log`).

## Commands

```bash
# Start everything
docker compose up -d

# Stop
docker compose down

# Stop AND wipe all data (fresh reset)
docker compose down -v && docker compose up -d -d

# Run a WP-CLI command
docker compose run --rm wpcli plugin list

# Re-run store setup (products, shipping, payment, options)
./setup.sh

# Diagnose common checkout load/submit causes (log, template, versions)
./diagnose-checkout.sh
```

## Reproducing the checkout load/submit error

1. `docker compose up -d`
2. `./setup.sh`
3. Add a product to the cart, open http://localhost:8085/checkout/
4. If the spinner never resolves or submit fails:
   - Check `./wp-content/debug.log` for PHP fatals/warnings.
   - Run `./diagnose-checkout.sh` to confirm theme/plugins/template.
   - Inspect the browser console/network tab on the checkout page
     (block-based checkout is JS-driven).

## Automated tests (Playwright)

`tests/` holds a Playwright suite that drives the real checkout in a browser
(Firefox by default, Chromium optional) and captures **JS console errors,
page errors, failed/HTTP>=400 requests, and screenshots** — the fastest way to
reproduce a "checkout fails to load / submit" report.

```bash
cd tests
npm install            # first time only (installs Playwright + browsers)
npm test               # run both tests on Firefox
npm run test:debug     # headed (watch it run live)
```

Test files: `tests/checkout.spec.js`
- **checkout page loads** — adds product to cart, opens `/checkout/`, asserts the
  block checkout hydrates with zero JS errors.
- **checkout form submits** — fills contact/address, clicks Place Order, reports
  `ORDER_SUCCESS`, `FINAL_URL`, and any console/page/HTTP errors.

Results (screenshots, videos, traces, JSON) land in `tests/test-results/`.

To target a different environment: `WOO_BASE_URL=https://example.com npm test`.

## Two bugs reproduced & fixed while setting this environment up

These are the classic "checkout page fails to load" and "can't submit" causes:

1. **`/checkout/` rendered the Blog instead of the checkout form.**
   Cause: the site was on **plain (query-string) permalinks**, so pretty URLs
   like `/checkout/`, `/shop/`, `/cart/` didn't resolve to their pages and fell
   through to the blog index. Fix: `wp rewrite structure '/%postname%/' --hard`
   (now in `setup.sh`). Symptom to look for on a client site: visiting `/checkout/`
   shows the blog/home page.

2. **Checkout form loads but "There are no payment methods available"**
   and the **Place Order button is disabled** → can't submit.
   Cause: no payment gateway was actually enabled (raw option writes didn't
   take; the gateway instance `enabled` flag was off). Fix: enable `cheque`/`cod`
   through the gateway settings API (now in `setup.sh`).

## Notes

- Because the checkout is the **block-based** checkout, most load/submit failures
  are JavaScript/asset/CORS related — check the browser Console and the
  `/wp-json/wc/store/checkout` requests in the Network tab.
- To test the legacy shortcode checkout, replace the checkout page content with
  `[woocommerce_checkout]`.
