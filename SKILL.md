---
name: cart-patrol
description: Run e2e tests on any e-commerce storefront. Tests navigation, product listing, product detail, cart management (add/remove/quantity), checkout form validation, and payment. Pass a URL to test any site, or omit to test the last-discussed site. Use when asked to test an e-commerce site or run cart-patrol.
---

# cart-patrol

Run e2e tests on any e-commerce site using vibium browser automation.

## How to run

`/cart-patrol [url] [flow]`

- `/cart-patrol https://example.com` — all flows on a URL
- `/cart-patrol https://example.com smoke` — one flow on a URL
- `/cart-patrol checkout` — one flow on the last-discussed site
- `/cart-patrol` — all flows on the last-discussed site

Flow names: `navigation`, `product`, `cart`, `checkout`, `smoke`, `all` (default)

Report pass/fail per flow with what you observed. Flag unexpected behavior as a bug. End with a results table: flow | result | notes.

---

## Generic test approach

For unknown sites, discover structure first, then test each flow. Prefer `vibium find text` and `vibium find role` over hardcoded `@refs` — refs expire on every page change.

### Flow 1 — Navigation
```
vibium go <url> && vibium map
# Identify nav links in header/nav
# Click each, verify URL changes and page loads
vibium url
vibium back && vibium wait load
```

### Flow 2 — Product listing
```
vibium go <catalog-url> && vibium text
# Verify: product names and prices visible
vibium eval 'document.querySelectorAll("button").length'
# Verify: at least one Add to Cart button
```

### Flow 3 — Product detail
```
# Click first in-stock product (check availability before clicking Add to Cart)
vibium click <product-link>
vibium wait load && vibium url
vibium text
# Verify: name, price, add-to-cart mechanism present
# Check if Add to Cart is disabled: vibium eval 'document.querySelector("button[title=\'Add to Cart\']").disabled'
# If disabled, check for required option dropdowns with no selectable values → bug
```

### Flow 4 — Add to cart
```
# Find and click Add to Cart
vibium find text "Add to Cart"
vibium click @e1
# If click fails with "obscured": vibium eval 'document.querySelector("button[title=\'Add to Cart\']").click()'
# Wait for confirmation (toast, badge update, or button text change)
vibium diff map
# Verify: cart count incremented OR button state changed
```

### Flow 5 — Cart management
```
# Navigate to cart via UI (badge or link — not direct URL on in-memory carts)
# Verify: item appears with correct price
# Update quantity:
#   - +/- buttons: click and verify subtotal updates
#   - qty input: fill new value, click Update, verify total
#   - qty input only (no update button): trigger change event via eval
# Remove: click remove/x, verify item gone and total updates
```

### Flow 6 — Empty cart
```
# Clear cart or use fresh session
# Verify: empty state message shown
# Navigate to checkout with empty cart
# Verify: blocked or redirected (no silent pass-through)
```

### Flow 7 — Checkout form validation
```
# Add item, reach checkout, submit without filling required fields
# Verify: inline errors shown for each required field
# Fill fields one by one and re-submit to confirm each error clears
```

### Flow 8 — Full checkout (happy path)
```
# Fill all required fields (billing, shipping, payment)
# On one-page checkouts: fill top-to-bottom; country triggers zone reload
#   → always re-check Terms checkbox after selecting country/state
# Submit and verify confirmation page
# Verify: order number or "Thank you" shown
# Verify: cart cleared after order
# Verify: success page NOT accessible without placing an order (navigate directly)
```

---

## Cross-site techniques

### Counting buttons
`vibium count "button"` has a JSON parse error on some sites:
```
vibium eval 'document.querySelectorAll("button").length'
```

### Clicking obscured or hard-to-reach buttons
When `vibium click` fails with "element is obscured" or "zero size", scroll into view first, or bypass via eval:
```
vibium scroll into-view "button[title='Add to Cart']" && vibium sleep 500
vibium click @eN
# If still failing:
vibium eval 'document.querySelector("button[title=\"Add to Cart\"]").click()'
```

### Disabled Add to Cart buttons
When Add to Cart is `disabled`, look for required option dropdowns:
```
vibium eval 'document.querySelectorAll("select").length'
vibium eval 'JSON.stringify([...document.querySelectorAll("select option")].map(o => o.textContent.trim()))'
```
If a required `<select>` has only a placeholder option and no real values, the product's options are broken — **report as a bug**, not a workaround. Find a different product to continue testing.

### Country/state dropdowns — numeric IDs (OpenCart)
OpenCart dropdowns use internal numeric IDs. Display names and ISO codes both fail silently. Always inspect first:
```
vibium eval 'JSON.stringify([...document.querySelector("#input-payment-country").options].filter(o=>o.text.includes("United States")).map(o=>({v:o.value,t:o.text})))'
# → [{"v":"223","t":"United States"}]
vibium select "#input-payment-country" "223"
vibium wait load && vibium sleep 500  # triggers zone reload
vibium eval 'JSON.stringify([...document.querySelector("#input-payment-zone").options].filter(o=>o.text.includes("Texas")).map(o=>({v:o.value,t:o.text})))'
# → [{"v":"3669","t":"Texas"}]
vibium select "#input-payment-zone" "3669"
```

### Country/state dropdowns — ISO codes (Shopify)
Shopify uses ISO codes, not display names:
```
vibium select @eN "US"    # country
vibium select @eN "TX"    # state
```
After selecting country, page reloads and clears address fields — refill them after `vibium wait load`.

### Checkbox reset after zone reload
On OpenCart, selecting country/state triggers a partial page reload that unchecks checkboxes (Terms & Conditions, newsletter). Always re-check after selecting zone:
```
vibium eval 'document.querySelector("[name=agree]").click()'
```
Do this as the last step before submitting, after all dropdowns are set.

### Obscured checkboxes
Styled checkboxes are often covered by a `<label>` element and fail with "obscured". Use eval:
```
vibium eval 'document.querySelector("[name=agree]").click()'
vibium eval 'document.querySelector("[name=account_agree]").click()'
```

### Hidden quantity inputs (Shopify)
Shopify carts render duplicate zero-size hidden quantity inputs. Filter for visible ones:
```
vibium eval 'const inputs = [...document.querySelectorAll("input[name=\"updates[]\"]")].filter(el => el.offsetWidth > 0); inputs[0].value = "2"; inputs[0].dispatchEvent(new Event("change")); "done"'
vibium find "input[name='update']" && vibium click @e1 && vibium wait load
```

### PCI payment iframes (Shopify)
Shopify payment fields are cross-origin iframes — `vibium fill` is blocked. Use mouse coordinates and single-character key presses:
```
vibium eval 'JSON.stringify(document.querySelector("[id^=card-fields-number]")?.getBoundingClientRect())'
vibium mouse click <center-x> <center-y>
vibium keys "1"  # type one character at a time — multi-char strings fail in iframes
```
Iframe IDs change per session. Discover them fresh:
```
vibium eval 'JSON.stringify([...document.querySelectorAll("iframe")].map((f,i) => ({i, id: f.id, src: f.src?.substring(0,60)})))'
```

### PrestaShop demo — ephemeral subdomains
`demo.prestashop.com` wraps the actual store in an iframe. Get the inner URL after a 5s wait, then navigate via JS — NOT `vibium go`:
```
vibium go https://demo.prestashop.com/ && vibium wait load && vibium sleep 5000
vibium eval 'document.querySelector("#framelive")?.src'
# → https://{subdomain}.demo.prestashop.com/en/...
vibium go https://{subdomain}.demo.prestashop.com/en/   # homepage only — see note below
```
**`vibium go` deadlocks on subdomain pages** (B3 pattern — daemon socket i/o timeout). Use JS navigation for all in-store page transitions:
```
vibium eval 'location.href = "https://{subdomain}.demo.prestashop.com/1-1-hummingbird-printed-t-shirt.html"'
vibium wait load --timeout 10000
```
The inner subdomain expires in ~2 min of active testing. If you get "Oops... couldn't find the shop", get a fresh subdomain and restart.

### PrestaShop qty input — Enter to commit
PrestaShop cart qty inputs require pressing Enter to trigger the update (fill alone does nothing):
```
vibium fill "input[name='product-quantity-spin']" "2"
vibium press Enter "input[name='product-quantity-spin']"
vibium sleep 2000
# Verify total updated
```

### PrestaShop remove — semantic selector required
`vibium find text "Remove"` returns an outer `<div>` that is not clickable. Target the `<a>` directly:
```
vibium find "a.js-remove-from-cart" && vibium click @e1 && vibium sleep 2000
# Reload to confirm — DOM update is async
```

### PrestaShop country ID — numeric values
PrestaShop uses numeric option values for countries, similar to OpenCart. Inspect before selecting:
```
vibium eval 'JSON.stringify([...document.querySelector("#field-id_country").options].map(o => ({v:o.value,t:o.text})))'
# → [{"v":"","t":"-- please choose --"},{"v":"8","t":"France"},{"v":"21","t":"United States"}]
vibium select "#field-id_country" "8"
```

### PrestaShop checkout — scroll fields into view before fill
Address form fields are off-screen and report zero-size until scrolled into view:
```
vibium scroll into-view "#field-address1" && vibium sleep 300
vibium fill "#field-address1" "123 Test Street"
```
Submit button also reports zero-size — use eval.click():
```
vibium eval 'document.querySelector("form button[type=submit]").click()'
```

### Cloudflare protection
`demo.nopcommerce.com` and `demo.opencart.com` are behind Cloudflare Turnstile (managed) and JS challenges respectively. These block headless browsers at the fingerprinting level — no DOM interaction can bypass them. Avoid these sites for automation testing.

### CSS text-transform — button text mismatch
Some sites render button text as uppercase via CSS (`text-transform: uppercase`) while the DOM contains mixed-case text. `vibium find text "ADD TO CART"` silently returns no match because the DOM contains `"Add to Cart"`:
```
vibium find text "ADD TO CART"   # → element not found (CSS, not DOM)
```
Solution: always use `vibium map` refs for buttons on affected sites. Affected sites: **AcademyBugs**, **QA Practice** (`qa-practice.razvanvancea.ro`).

### SPA in-memory carts — never use vibium go for cart navigation
Some SPAs store cart state in memory (Vue/React component state). A `vibium go` to the cart URL causes a full page reload that wipes the cart. Always navigate via UI links:
```
vibium find "a[href='/cart']" && vibium click @e1 && vibium wait load
# NOT: vibium go https://coffee-cart.app/cart
```
Sites with in-memory carts: **var.parts**, **coffee-cart.app**

### coffee-cart.app — product cards are divs, not buttons
Products are `div.cup-body[data-test][aria-label]` elements. Use `data-test` attribute to target them:
```
vibium find "[data-test='Espresso']" && vibium click @e1
vibium find "[data-test='Flat_White']" && vibium click @e1   # underscores for spaces
```
Right-click opens a confirm dialog — dispatch contextmenu event:
```
vibium eval 'document.querySelector("[data-test=\"Espresso\"]").dispatchEvent(new MouseEvent("contextmenu", {bubbles:true}))'
vibium sleep 500
vibium find role button --name "Yes" && vibium click @e1   # or "No" to dismiss
```

### Web Components shadow DOM — Polymer Shop
`vibium map` returns "No interactive elements found" on any Polymer Shop page. All UI lives in nested `<shop-app>` custom element shadow DOMs. Pattern for every interaction:
```
# 1. Find element via shadow DOM traversal
vibium eval 'JSON.stringify(document.querySelector("shop-app").shadowRoot.querySelector("shop-detail")?.shadowRoot?.querySelector("shop-button")?.getBoundingClientRect())'
# → {"x":720,"y":734,"width":186,"height":36,...}

# 2. Click via coordinates
vibium mouse click 813 752

# 3. Read state via eval
vibium eval 'document.querySelector("shop-app").shadowRoot.querySelector("shop-cart")?.shadowRoot?.querySelector(".subtotal")?.textContent?.trim()'
```

Shadow DOM component map:
- `shop-app > shop-list` — product listing
- `shop-app > shop-detail` — product detail (has `shop-button`, size/qty selects)
- `shop-app > shop-cart` — cart (has `shop-cart-item`, total, checkout button)
- `shop-app > shop-checkout` — checkout form (accountEmail, accountPhone, ship*, cc*)
- `shop-app > shop-cart-button` — header cart badge

Navigation links are reachable via direct URL — use `vibium go` to navigate between pages. Cart is server-side.

### Cart state persistence
- **var.parts**: in-memory — never navigate directly to `/cart`. Use the cart badge in the header.
- **coffee-cart.app**: in-memory — use `vibium find "a[href='/cart']" && vibium click @e1`, not `vibium go`.
- **Shopify**: server-side session — direct `/cart` URL is safe.
- **saucedemo.com**: server-side session — direct `/cart.html` URL is safe.
- **OpenCart (lambdatest)**: server-side session — direct `/cart` URL is safe.
- **Polymer Shop**: server-side session — direct `/cart` URL is safe.
- **AbanteCart (automationteststore)**: server-side session — direct `/index.php?rt=checkout/cart` safe.
- **PrestaShop (demo)**: server-side session per ephemeral subdomain — direct `/cart?action=show` is safe within the same subdomain, but subdomains expire in ~2 min of active testing.

### AbanteCart — Add to Cart is an anchor, not a button
AbanteCart uses `<a class="cart">` for Add to Cart, not `<button>`. `vibium find role button` will miss it:
```
vibium find "a.cart" && vibium click @e1
# On click, page navigates directly to cart (no toast or modal)
```
Some products are "Call To Order" type with no `a.cart` — skip them, pick a different product.

### AbanteCart — qty input name contains a hash
The quantity input name is `quantity[productid:hash]`, not a simple `name="quantity"`. Discover it via eval:
```
vibium eval 'JSON.stringify([...document.querySelectorAll("input")].filter(i => i.name.startsWith("quantity")).map(i => ({name: i.name, value: i.value})))'
# → [{"name":"quantity[53:b1a0e11451071a263d5a530074cc3395]","value":"1"}]
vibium fill "input[name='quantity[53:b1a0e11451071a263d5a530074cc3395]']" "3"
vibium find "button[title='Update']" && vibium click @e1 && vibium wait load
```

### AbanteCart — remove via URL parameter
Remove link is an `<a>` with `href` containing `remove=productid:hash`:
```
vibium find "a[href*='remove=']" && vibium click @e1 && vibium wait load
```

### Hibernating backends (Azure / Heroku)
Demo sites hosted on Azure App Service or Heroku free tier hibernate after inactivity. If the product listing loads with zero items and no error:
```
vibium sleep 5000 && vibium reload && vibium wait load --timeout 20000
```
Wait up to 30 seconds total. If products still don't appear after one reload, log as infrastructure issue (not an app bug) and skip the site.

### Daemon stability
On long test runs the vibium daemon can drop. If any command fails with "broken pipe" or "i/o timeout":
```
vibium daemon stop && sleep 2 && vibium daemon start && sleep 2
# Then re-navigate and log in if needed
```

---

## Site profiles

### var.parts (https://var.parts/)

- **Platform:** Custom React, in-memory cart
- **Products:** 12 items — SWAG, ACCESSORIES, HEAD UNITS, CHASSIS, WIRING, HARDWARE
- **Cart:** In-memory — never use `vibium go https://var.parts/cart`. Always navigate via the cart badge.
- **Add to cart:** Toast "Added to cart [name]", button changes to "In Cart"
- **Checkout fields:** Unit Designation + Service Bay (both required)
- **Validation error:** "Missing address" + "Unit designation and service bay are required."
- **Shipping:** Lunar (FREE, default) or Interplanetary Express (⚡149.99)
- **Payment:** Two tabs — "Pay Here" (Pay Now button) and "Other Device" (QR code + copy link)
- **Post-payment:** "Payment Received!", cart cleared, "Back to Shop" → `/`
- **Copy link feedback:** "Link copied to clipboard" (not "Copied")
- **Test data:** Unit Designation `VAR-402`, Service Bay `Bay 14-C`

---

### sauce-demo.myshopify.com (https://sauce-demo.myshopify.com/)

- **Platform:** Shopify
- **Products:** 7 total, 2 sold out (Brown Shades, White sandals), currency GBP
- **Catalog:** `/collections/all`
- **Cart:** Server-side session — direct `/cart` URL safe
- **Variants:** Noir jacket has size (S/M/L) + color (Blue/Red) — select both before Add to Cart
- **Quantity update:** Filter visible `input[name="updates[]"]` by `offsetWidth > 0`, dispatch `change` event, then click `input[name="update"]`
- **Checkout:** Email → delivery → shipping (loads dynamically after full address) → PCI payment
- **Country/state:** ISO codes only — `US`, `TX`; selecting country reloads and clears address fields
- **Payment:** PCI iframes from `checkout.pci.shopifyinc.com`; use mouse coords + single `vibium keys` chars
- **Iframe IDs:** `card-fields-number-*`, `card-fields-expiry-*`, `card-fields-verification_value-*`, `card-fields-name-*`
- **Test payment:** card `1`, expiry `12/30`, CVV `123`, name `Tester`
- **Post-payment:** `/thank-you` page with order number, summary, and shipping details

---

### saucedemo.com (https://www.saucedemo.com)

- **Platform:** Custom React, login-gated demo
- **Login:** `standard_user` / `secret_sauce` (credentials shown on login page)
- **Products:** 6, all in stock, USD
- **Cart:** Server-side session — direct `/cart.html` URL safe
- **Navigation:** Burger menu only (`#react-burger-menu-btn`) — always close before re-opening
- **Add to cart:** Button text changes to "Remove", cart badge increments
- **Cart:** No quantity controls — add/remove only
- **Checkout:** Info form → overview → complete (no payment form; fake "SauceCard #31337")
- **Post-order:** `/checkout-complete.html` "Thank you for your order!", cart cleared
- **Known bugs:**
  - Empty cart checkout not blocked — proceeds to form with no warning
  - "About" menu link navigates to external site that hangs (recover with `vibium go`)
- **Test data:** First Name `Test`, Last Name `User`, Postal Code `12345`
- **Burger menu pattern:**
  ```
  vibium click "#react-burger-menu-btn" && vibium sleep 500
  # interact
  vibium click "#react-burger-cross-btn" && vibium sleep 600
  ```

---

### demo.prestashop.com (https://demo.prestashop.com/)

- **Platform:** PrestaShop
- **Access pattern:** Main URL wraps the store in an iframe. Get the inner URL via `vibium eval 'document.querySelector("#framelive")?.src'` after a 5s wait. Navigate to the homepage with `vibium go` — but use `eval 'location.href = "..."'` + `vibium wait load` for all subsequent in-store navigation.
- **Ephemeral subdomains:** Each instance expires in ~2 min of active testing. If the page shows "Oops... couldn't find the shop", get a fresh subdomain and restart.
- **`vibium go` deadlocks on subdomain pages (B3):** Any `vibium go` to a subdomain page causes daemon socket i/o timeout. Only `vibium go` to the homepage on first load is safe. All further navigation must use JS:
  ```bash
  vibium eval 'location.href = "$STORE/en/3-clothes"'  # not: vibium go or vibium click nav
  vibium wait load --timeout 10000
  vibium eval 'location.href = "$STORE/1-1-hummingbird-printed-t-shirt.html"'
  vibium wait load --timeout 10000
  ```
- **Products:** Hummingbird printed t-shirt (€22.94 sale / €28.68 regular), Brown bear sweater (€34.46), framed posters. Categories: Clothes, Accessories, Art.
- **Product URL pattern:** `/{subdomain}/1-1-hummingbird-printed-t-shirt.html`, `/2-9-brown-bear-printed-sweater.html`
- **Product detail variant selectors:** Size = `#input_1_1` (`name="group[1]"`, values `1`–`4` for S/M/L/XL); Color = `input[name="group[2]"]` (val=`8` White, val=`11` Black)
- **Cart:** Server-side session per subdomain — direct `/cart?action=show` is safe within same instance
- **Add to cart:** Modal appears with "Continue shopping" and "Proceed to checkout"; cart badge shows "View cart (N products)"
- **Checkout:** 4-step: Personal Information → Addresses → Shipping method → Payment
- **Shipping options:** Click and collect (Free), My carrier (€8.40)
- **Payment options:** Pay by bank wire, Cash on Delivery, Pay by Check (no real payment gateway)
- **Country dropdown:** Numeric IDs — inspect options: `8` = France, `21` = United States
- **Address fields:** Off-screen until scrolled into view; use `vibium scroll into-view "#field-address1"` before filling
- **Submit buttons:** Often zero-size to vibium — use `vibium eval 'document.querySelector("form button[type=submit]").click()'`
- **Remove from cart:** Target `a.js-remove-from-cart` not the outer div; reload to confirm removal
- **Qty update:** Fill input + press Enter (fill alone does not trigger update)
- **Post-order:** No test yet (ephemeral subdomain expired before completion)
- **Test data:** First `Test`, Last `User`, Email `testuser@example.com`, Address `123 Test Street`, Postcode `75001`, City `Paris`, Country `8` (France)
- **Known bugs:**
  - **Add to cart button permanently disabled** — `[data-button-action=add-to-cart]` becomes `disabled=true` after PrestaShop JS initializes on page load. Affects product detail pages and homepage product cards. Click via map ref succeeds without error but AJAX fails silently and cart stays at 0. Direct AJAX POST to `/cart` returns HTML error page, not JSON. Checkout is unreachable because cart cannot be populated.
  - Cart qty increase button (`+`) in cart is also permanently `disabled`
  - Empty cart checkout (`/en/order`) redirects to `/cart?action=show` showing "There are no more items in your cart" — correct behavior, expected
  - "Your address is incomplete" error may appear on valid addresses (demo instance data isolation issue)

---

### coffee-cart.app (https://coffee-cart.app/)

- **Platform:** Custom Vue.js SPA, in-memory cart
- **Products:** 9 coffees — Espresso $10, Espresso Macchiato $12, Cappuccino $19, Mocha $8, Flat White $18, Americano $7, Cafe Latte $16, Espresso Con Panna $14, Cafe Breve $15
- **Cart:** In-memory — never use `vibium go https://coffee-cart.app/cart`. Navigate via `vibium find "a[href='/cart']" && vibium click @e1`.
- **Product cards:** `div.cup-body[data-test][aria-label]` — click to add directly; use `data-test` attribute (underscores for spaces: `Flat_White`, `Cafe_Latte`, `Espresso_Con_Panna`, `Cafe_Breve`)
- **Right-click dialog:** Dispatching `contextmenu` event opens "Add X to the cart?" modal with Yes/No buttons
- **Cart controls:** `button[aria-label="Add one X"]`, `button[aria-label="Remove one X"]`, `button[aria-label="Remove all X"]` — totals update instantly
- **Empty cart:** "No coffee, go add some." message; checkout button disappears entirely
- **Checkout:** Modal overlay (not a page) — fields `#name`, `#email`, optional `#promotion` checkbox
- **Validation:** HTML5 native — name required, email required + format checked
- **Post-order:** "Thanks for your purchase. Please check your email for payment." banner; cart cleared; random product gets gold promo highlight
- **No order confirmation URL** — success is an on-page banner only
- **Special behaviors:**
  - Promo popup on every 3rd item added: "It's your lucky day! Get an extra cup of [X] for $4." — Yes adds at discount, "Nah, I'll skip." dismisses
  - Double-click `<h4>` title translates name to Chinese (e.g. 特浓咖啡); double-click again reverts
  - `?breakable=1` param simulates intermittent add-to-cart errors (random, not every click)
  - `?ad=1` param adds ad banners that slow page load

---

### ecommerce-playground.lambdatest.io (https://ecommerce-playground.lambdatest.io/)

- **Platform:** OpenCart
- **Products:** 75+ per category; many products have broken required option selects (placeholder only, no values) — Add to Cart disabled with no path to resolution
- **Reliable test product:** HP LP3065 (`product_id=47`) — in stock, no required options, $122.00
- **Cart:** Server-side session — direct `/index.php?route=checkout/cart` URL safe
- **Add to cart:** Toast "Success: You have added [product] to your shopping cart!" with "View Cart →" and "Checkout →" buttons; navigate to cart via `vibium go` on the cart URL (toast buttons disappear quickly)
- **Cart controls:** `button[title="Update"]` and `button[title="Remove"]`; qty is a number input
- **Checkout:** One-page form (guest or register) → confirm order page → success
  - Guest checkout: `vibium eval 'document.querySelector("[name=account][value=guest]").click()'`
  - Country uses numeric IDs (e.g. `223` = United States); state uses numeric IDs (e.g. `3669` = Texas)
  - Selecting country triggers partial page reload — refill address fields and re-check Terms after
  - Terms & Conditions checkbox is obscured by label — use `vibium eval 'document.querySelector("[name=agree]").click()'`
  - Submit via `vibium eval 'document.querySelector("#button-save").click()'`
- **Payment:** Cash on Delivery (default), Flat Shipping Rate ($5.00)
- **Post-order:** `/index.php?route=checkout/success` — "Your order has been successfully processed!"
- **Known bugs:**
  - `/checkout/success` accessible directly without placing an order — shows confirmation with no order context
  - Many products have required option dropdowns with placeholder only — blocks Add to Cart entirely
  - Country/state zone reload resets Terms & Conditions checkbox
  - Post-order "Continue" button loops back to the same success URL instead of redirecting home
- **Test data:** First Name `Test`, Last Name `User`, Email `guest_<timestamp>@example.com`, Tel `12025550100`, Address `123 Test Street`, City `Austin`, Post Code `78701`, Country `223` (US), Zone `3669` (Texas)

---

### automationteststore.com (https://automationteststore.com/)

- **Platform:** AbanteCart (OpenCart fork)
- **Products:** Apparel & Accessories, Makeup, Skincare, Fragrance, Men, Hair Care, Books categories; some products are "Call To Order" type with no Add to Cart
- **Reliable test product:** Tropiques Minerale Loose Bronzer (`product_id=53`, Makeup) — $38.50, Color dropdown, Add to Cart present
- **Cart:** Server-side session — direct `/index.php?rt=checkout/cart` URL safe
- **Add to cart:** `<a class="cart">` (not a button) — click navigates directly to cart page (no toast or modal)
- **Cart qty input:** Name pattern `quantity[productid:hash]` — inspect via eval to get exact name before filling
- **Cart controls:** `button[title="Update"]` for qty, `a[href*='remove=']` for removal
- **Checkout:** Guest or register option → guest_step_1 (billing/shipping form) → auto-skips step 2 when one option → checkout confirmation → success
  - Guest: `vibium eval 'document.querySelector("input[name=account][value=guest]").click()'` then click Continue
  - Country/state use same numeric IDs as OpenCart: `223` = US, `3669` = Texas
  - No partial page reload after country select — no field clearing, no Terms reset
  - Tax (8.5%) added at confirmation stage
- **Payment:** Cash on Delivery (default), Flat Shipping Rate ($2.00)
- **Post-order:** `/index.php?rt=checkout/success` — "YOUR ORDER HAS BEEN PROCESSED!"; cart cleared
- **Success page security:** Direct navigation to success URL redirects to homepage (correctly secured — no order context leak)
- **No known bugs**
- **Test data:** First Name `Test`, Last Name `User`, Email `guest_test@example.com`, Tel `12025550100`, Address `123 Test Street`, City `Austin`, Post Code `78701`, Country `223` (US), Zone `3669` (Texas)

---

### AcademyBugs (https://academybugs.com/)

- **Platform:** Custom PHP/HTML, bug-hunting practice site — 25 planted bugs
- **Purpose:** Bug discovery, not real e-commerce. All cart behaviors may be intentionally broken.
- **Setup:** Dismiss the tutorial modal (× button in top-right of overlay) AND the cookie banner ("Accept cookies") before interacting — both obscure elements and block clicks.
- **Products:** Apparel and accessories; prices in USD
- **Cart flow:** Works partially; product detail page has a planted layout bug — adding to cart from product detail causes the cart dropdown to render with broken/overlapping layout
- **Sort/filter:** Sort dropdown on listing page is non-functional (intentional bug — no reorder occurs). "View 12/24" filter buttons are unresponsive.
- **CSS text-transform:** Nav items and some buttons are rendered uppercase via CSS but have mixed-case DOM text — use actual DOM text with `vibium find text`, not the uppercase display text
- **Known bugs (cart-relevant):**
  - Sort dropdown has no effect — products remain in original order regardless of selection
  - View count filter (12/24) does not change number of visible products
  - Add to Cart from product detail page breaks cart dropdown layout
- **Note:** Not suitable for full checkout flow testing — cart bugs are intentional and checkout is not a test target here

---

### BookCart (https://bookcart.azurewebsites.net/)

- **Platform:** Angular SPA, Azure App Service backend
- **Hibernation:** Azure backend hibernates after inactivity. On first load, product list may be empty. Wait 5–10s and reload once — backend wakes within ~30s. If products still don't appear, skip; log as infrastructure issue.
- **Products:** Books across multiple categories; prices in USD; displayed after backend wakes
- **Cart:** Server-side session — direct cart URL safe once backend is awake
- **Login:** `admin` / `admin` (default credentials documented on site's About page); silent failure on wrong credentials — no error message shown
- **Registration:** Completes with no email confirmation and no success message — only redirect
- **Add to cart:** Requires login; if not logged in, add-to-cart may silently fail or redirect to login
- **Known bugs:**
  - Login with invalid credentials shows no error feedback — silent failure
  - Registration provides no confirmation to the user
  - Product API call fails silently on cold start — no loading state or error shown to user

---

### Magento (https://magento.softwaretestingboard.com/)

- **Status: DOWN as of 2026-04-22** — Cloudflare 526 SSL certificate error. Site is unreachable.
- Do not attempt to test. Log as infrastructure failure and skip.

---

### Polymer Shop (https://shop.polymer-project.org/)

- **Platform:** Polymer Web Components (v1/v2 era)
- **Critical limitation:** `vibium map` returns "No interactive elements found" on every page — all UI is inside nested shadow DOMs (`shop-app > shop-list/detail/cart/checkout`). Normal vibium commands (click, fill, find) cannot target anything.
- **Interaction pattern:** Use `eval + shadowRoot traversal` to find elements → `getBoundingClientRect()` for coordinates → `vibium mouse click x y`
- **Products:** 4 categories (Men's/Ladies Outerwear, Men's/Ladies T-Shirts); ~16 items per category; prices in USD
- **Cart:** Server-side session — direct `/cart` URL is safe
- **Add to cart:** Click `shop-button` inside `shop-detail` shadow DOM; confirm via `shop-cart-item` count in cart shadow DOM
- **Checkout fields:** accountEmail (required), accountPhone (required), shipAddress/City/State/Zip (required), ccName/ccNumber/ccCVV (required); setBilling checkbox for separate billing address
- **Checkout validation:** 9 fields show `input:invalid` on empty submit — HTML5 native validation
- **No real payment processing** — demo only
- **Navigation:** Use `vibium go` with direct URLs; shadow DOM links are reachable
- **Known shadow DOM component map:**
  - `shop-app.shadowRoot.querySelector("shop-list")` — product listing
  - `shop-app.shadowRoot.querySelector("shop-detail")` — product detail + Add to Cart
  - `shop-app.shadowRoot.querySelector("shop-cart")` — cart + total + checkout button
  - `shop-app.shadowRoot.querySelector("shop-checkout")` — checkout form

---

### practicesoftwaretesting.com (https://practicesoftwaretesting.com/)

- **Platform:** Angular SPA (Toolshop — hand tools, power tools, rentals)
- **Products:** Hand tools, power tools, other, special tools, rentals; paginated (5 pages); brand and category filters
- **Cart:** Server-side session — add to cart works without login; login required only at checkout step 2
- **Search:** Works — `vibium find placeholder "Search"` + fill + `vibium find role button --name "Search"` + click
- **Filters:** Category and brand checkboxes work — `vibium find "input[name=brand_id]"` + `vibium check @e1`
- **Login:** `customer@practicesoftwaretesting.com` / `welcome01` — **may be locked** (shared account locked by previous sessions); admin credentials not publicly documented
- **Login submit:** `input[type=submit]` not `<button>` — `vibium find role button --name "Login"` fails with timeout; use `vibium eval 'document.querySelector("input[type=submit][value=Login]").click()'`
- **Checkout wizard:** 4 steps — Cart → Sign in → Billing Address → Payment; step indicators clearly labeled
- **API:** REST API + Swagger at https://api.practicesoftwaretesting.com/api/documentation
- **Angular async delay:** After applying category/brand filters, DOM updates are async — add `vibium sleep 1500` before reading results
- **Known issues:**
  - Shared test account locks after too many failed login attempts — no self-service unlock
  - Search for "pliers" returns "Leather toolbelt" — possible keyword/tag match, not strict relevance

### Test flows

#### Smoke test
```bash
# Browse products
vibium go https://practicesoftwaretesting.com/ && vibium wait load
vibium text
# Verify: products with names and prices visible

# Add to cart without login
vibium map
# Find a product link — e.g. @e5 [a] "Slip Joint Pliers"
vibium click @e5 && vibium wait load
vibium find text "Add to cart" && vibium click @e1 && vibium sleep 1000
vibium eval 'document.querySelector(".navbar [class*=cart]")?.textContent?.trim()'
# Verify: cart badge incremented

# Search
vibium go https://practicesoftwaretesting.com/ && vibium wait load
vibium find placeholder "Search" && vibium fill @e1 "hammer"
vibium find role button --name "Search" && vibium click @e1 && vibium sleep 1500
vibium text
# Verify: results contain hammer-related products
```

#### Category filter
```bash
vibium go https://practicesoftwaretesting.com/ && vibium wait load
vibium find "input[name=by_category]" && vibium check @e1 && vibium sleep 1500
vibium text
# Verify: product list narrowed to selected category
```

#### Login (if account not locked)
```bash
vibium go https://practicesoftwaretesting.com/auth/login && vibium wait load
vibium fill "#email" "customer@practicesoftwaretesting.com"
vibium fill "#password" "welcome01"
vibium eval 'document.querySelector("input[type=submit][value=Login]").click()' && vibium sleep 2000
vibium url
# Expected: / (redirected to home on success)
# If account locked: error shown on login page — no self-service unlock available
```

---

### qa-practice.razvanvancea.ro (https://qa-practice.razvanvancea.ro/)

- **Platform:** Custom HTML/JS, multi-section QA practice site (moved from qa-practice.netlify.app)
- **Login:** `admin@admin.com` / `admin123` (credentials shown as placeholder text on login page)
- **Session:** NOT persistent — page reload wipes cart and logs out (localStorage/in-memory)
- **Products:** 200 items (cosmetics), paginated — 10 per page, 20 pages
- **Add to cart:** Buttons use CSS `text-transform: uppercase` — DOM text is `"ADD TO CART"`. `vibium find text` fails. Use `vibium map` refs: after login on page 1, product buttons are `@e27`–`@e36`.
- **Cart:** Shown as a table at top of page; updates in-place when items added
- **Checkout:** "PROCEED TO CHECKOUT" button toggles DOM — hides product section, reveals Shipping Details form (Phone, Street, City, Country). Form parent starts as `display:none`.
- **Post-order:** Inline confirmation on same page — no separate URL, no order number. Message: "Congrats! Your order of $X.XX has been registered and will be shipped to [address]."
- **Cart cleared on checkout:** After clicking PROCEED TO CHECKOUT, cart visually resets to $0 — expected (products hidden, form shown), not a bug.

### Test flows

#### Smoke test
```bash
# Login
vibium go https://qa-practice.razvanvancea.ro/auth_ecommerce.html && vibium wait load
vibium map
# @e25 = email, @e26 = password, @e27 = submit
vibium fill @e25 "admin@admin.com" && vibium fill @e26 "admin123" && vibium click @e27
vibium wait load
# Verify: Log Out link visible, Shopping Cart heading, product grid loaded

# Add to cart (use map refs — CSS uppercase breaks find text)
vibium map
# @e27 = first ADD TO CART button
vibium click @e27 && vibium sleep 500
vibium eval 'document.querySelector(".table")?.textContent?.trim()?.slice(0,80)'
# Verify: product name appears in cart table with price

# Proceed to checkout
vibium map
# @e26 = PROCEED TO CHECKOUT button
vibium click @e26 && vibium sleep 500
vibium map --selector "form"
# Verify: Phone, Street, City, Country, Submit Order fields appear

# Fill and submit order
vibium fill @e1 "1234567890"  # phone
vibium fill @e2 "123 Main St"  # street
vibium fill @e3 "New York"     # city
vibium select @e4 "United States of America"
vibium click @e5 && vibium sleep 500
vibium text
# Verify: "Congrats! Your order of $..."
```
