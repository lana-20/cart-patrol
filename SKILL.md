---
name: cart-patrol
description: Run e2e tests on any e-commerce storefront. Tests navigation, product listing, product detail, cart management (add/remove/quantity), checkout form validation, and payment. Pass a URL to test any site, or omit to test the last-discussed site. Use when asked to test an e-commerce site or run cart-patrol.
---

# cart-patrol

Run e2e tests on any e-commerce site using vibium browser automation.

## How to run

ARGUMENTS format: `[url] [flow]`

- `/cart-patrol https://example.com` — run all flows on a given URL
- `/cart-patrol https://example.com smoke` — run smoke test only
- `/cart-patrol checkout` — run checkout flow on the already-known site
- `/cart-patrol` — run all flows on the last-discussed or most recently tested site

Supported flow names: `navigation`, `product`, `cart`, `checkout`, `smoke`, `all` (default)

For each flow, report pass/fail with what you observed. Note unexpected behavior as a bug. End with a results table: flow name | pass/fail | notes.

## Generic test approach

For unknown sites, discover the structure first, then test each flow. Use semantic finders (`vibium find text`, `vibium find role`) rather than hardcoded `@refs` — refs shift between sites and page states.

### Flow 1: Navigation

```
vibium go <url> && vibium map
# Find nav links (a elements in header/nav)
# Click each nav link, verify URL changes and page loads
vibium url  # confirm expected route
vibium back && vibium wait load
```

### Flow 2: Product listing

```
vibium go <catalog-url> && vibium text
# Verify: product names and prices visible
vibium eval 'document.querySelectorAll("button").length'
# Verify: at least one button (Add to Cart or equivalent)
```

### Flow 3: Product detail

```
# Click first available (non-sold-out) product link
vibium map
vibium click <product-link-ref>
vibium wait load && vibium url
# Verify: URL changed to product page
vibium text
# Verify: product name, price, and add-to-cart mechanism present
```

### Flow 4: Add to cart

```
# Find and click Add to Cart
vibium find text "Add to Cart"
vibium click @e1
# Wait for confirmation (toast, badge update, or button text change)
vibium diff map
# Verify: cart badge/count incremented OR button changed state
```

### Flow 5: Cart management

```
# Navigate to cart via UI (badge, link, or /cart)
# Verify: added item(s) appear with correct price
# Update quantity:
#   - Sites with +/- buttons: click them, verify subtotal updates
#   - Sites with qty input: fill new value, click Update, verify total
# Remove item: click remove/x/delete, verify item gone and total updates
```

### Flow 6: Empty cart

```
# After clearing cart or in fresh session:
# Verify: empty state message shown ("Your cart is empty" or equivalent)
# Navigate to checkout with empty cart:
# Verify: blocked or shows empty-cart message
```

### Flow 7: Checkout form validation

```
# Add item, reach checkout, submit without filling required fields
vibium find role button --name "checkout"  # or site-specific button
# Verify: validation errors shown for required fields
```

### Flow 8: Full checkout (happy path)

```
# Fill all required delivery fields
# Select shipping method if prompted
# Fill payment fields
# Submit and verify confirmation page
# Verify: order number or "Thank you" confirmation shown
# Verify: cart cleared after order
```

## Cross-site techniques

### Counting buttons safely
`vibium count "button"` has a known JSON parse error on some sites. Use eval instead:
```
vibium eval 'document.querySelectorAll("button").length'
```

### Hidden quantity inputs
Some carts (Shopify) render duplicate hidden qty inputs. Filter for visible ones:
```
vibium eval 'const inputs = [...document.querySelectorAll("input[name=\"updates[]\"]")].filter(el => el.offsetWidth > 0); inputs[0].value = "2"; inputs[0].dispatchEvent(new Event("change")); "done"'
```

### PCI payment iframes (Shopify)
Shopify payment fields are cross-origin iframes — `vibium fill` is blocked. Use:
```
vibium eval 'JSON.stringify(document.querySelector("#<iframe-id>")?.getBoundingClientRect())'
vibium mouse click <x> <y>   # click iframe center
vibium keys "<digit>"         # type one character at a time (no multi-char strings)
```

### Country/state dropdowns with ISO codes
Shopify checkout dropdowns use ISO codes, not display names. `vibium select @e7 "United States"` silently fails. Use:
```
vibium select @e7 "US"   # country
vibium select @e14 "TX"  # state
```
After selecting country, page may reload and clear address fields — refill them after `vibium wait load`.

### Cart state persistence
- **var.parts**: in-memory only — cart lost on page reload or direct `/cart` navigation. Always navigate to cart via UI (cart badge).
- **Shopify**: server-side session — `/cart` URL works regardless of how you navigate.
- **saucedemo.com**: server-side session — direct URL navigation safe.

### Login-gated sites (saucedemo.com)
Always log in before testing any flow. Credentials for saucedemo.com are shown on the login page itself:
```
vibium fill "#user-name" "standard_user"
vibium fill "#password" "secret_sauce"
vibium click "#login-button"
vibium wait load
```

### Burger menu navigation (saucedemo.com)
The menu button is obscured while the menu is open. Always close before re-opening:
```
vibium click "#react-burger-menu-btn" && vibium sleep 500
# interact with menu items
vibium click "#react-burger-cross-btn" && vibium sleep 600
```

## Site profiles

### var.parts (https://var.parts/)

- 12 products, categories: SWAG, ACCESSORIES, HEAD UNITS, CHASSIS, WIRING, HARDWARE
- Cart: in-memory — never `vibium go https://var.parts/cart` after adding items
- Add to cart confirmation: toast "Added to cart [name]", button changes to "In Cart"
- Required checkout fields: Unit Designation, Service Bay
- Validation error: "Missing address" + "Unit designation and service bay are required."
- Shipping options: Lunar (FREE, default), Interplanetary Express (⚡149.99)
- Payment UI: custom, two tabs — "Pay Here" (Pay Now button) and "Other Device" (QR + copy link)
- Copy link feedback: "Link copied to clipboard" (not "Copied")
- Post-payment: "Payment Received!" confirmation, cart cleared, "Back to Shop" → /
- Test data: Unit Designation `VAR-402`, Service Bay `Bay 14-C`

### sauce-demo.myshopify.com (https://sauce-demo.myshopify.com/)

- 7 products (2 sold out: Brown Shades, White sandals), currency GBP
- Cart: Shopify server-side session — `/cart` URL is safe to navigate directly
- Catalog at `/collections/all`
- Noir jacket has size (S/M/L) and color (Blue/Red) variant selectors — select variant before "Add to Cart"
- Quantity update: fill visible `input[name="updates[]"]` (filter by offsetWidth > 0) + click `input[name="update"]`
- Checkout: standard Shopify flow — email → delivery → shipping → payment
- Country/state: ISO codes required (`US`, `TX`) — display names silently fail
- Shipping: loads dynamically after full address entry; International Shipping £20
- Payment: PCI iframes from `checkout.pci.shopifyinc.com` — use mouse coordinates + `vibium keys` one char at a time
- Iframe IDs: `card-fields-number-*`, `card-fields-expiry-*`, `card-fields-verification_value-*`, `card-fields-name-*`
- Test payment: card `1`, expiry `12/30`, CVV `123`, name `Tester`
- Shopify bogus gateway accepts card number `1` — full order confirmation with order # returned
- Post-payment: `/thank-you` page with confirmation number, order summary, and shipping details

### saucedemo.com (https://www.saucedemo.com)

- Login required — credentials shown on login page; use `standard_user` / `secret_sauce`
- 6 products, all available (no sold-out items for standard_user), currency USD
- Cart: server-side session — direct `/cart.html` URL navigation is safe
- Navigation via burger menu (`#react-burger-menu-btn`) — not a standard nav bar
- Add to cart confirmation: button text changes to "Remove", cart badge increments
- No quantity controls in cart — add/remove only, qty shown as static label
- Checkout: info form (first name, last name, postal code) → overview → complete
- Payment: no payment form — fake "SauceCard #31337" applied automatically
- Post-order: `/checkout-complete.html` with "Thank you for your order!", cart cleared
- Known bug: empty cart checkout not blocked — proceeds to checkout form with no warning
- Known bug: "About" menu link navigates to external site that may hang (use `vibium go` to recover)
- Test data: First Name `Test`, Last Name `User`, Postal Code `12345`
