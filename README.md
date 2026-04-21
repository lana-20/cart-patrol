# cart-patrol

Site-agnostic e2e test skill for e-commerce storefronts. Tests navigation, product listing, product detail, cart management, checkout form validation, and payment flows. Built on [vibium](https://www.npmjs.com/package/vibium) browser automation.

## Installation

**Via skills CLI (recommended):**
```bash
npx skills add lana-20/cart-patrol -g
```

**Manual:**
```bash
git clone https://github.com/lana-20/cart-patrol.git ~/.claude/skills/cart-patrol
```

Requires [vibium](https://www.npmjs.com/package/vibium):
```bash
npm install -g vibium
```

## Usage

```
/cart-patrol <url>                  # run all flows on any site
/cart-patrol <url> <flow>           # run a specific flow
/cart-patrol <flow>                 # run a flow on the last-discussed site
/cart-patrol                        # run all flows on the last-discussed site
```

**Examples:**
```
/cart-patrol https://var.parts/
/cart-patrol https://ecommerce-playground.lambdatest.io/ cart
/cart-patrol smoke
```

**Flow names:** `navigation`, `product`, `cart`, `checkout`, `smoke`, `all`

---

## Tested sites

| Site | Platform | Cart | Payment | Notes |
|------|----------|------|---------|-------|
| [var.parts](https://var.parts/) | Custom React | In-memory | Custom (fake) | Cart lost on direct URL navigation |
| [sauce-demo.myshopify.com](https://sauce-demo.myshopify.com/) | Shopify | Server-side | PCI iframes | 2 sold-out products |
| [saucedemo.com](https://www.saucedemo.com) | Custom React | Server-side | Fake (auto) | Login-gated demo, no qty controls |
| [ecommerce-playground.lambdatest.io](https://ecommerce-playground.lambdatest.io/) | OpenCart | Server-side | Cash on Delivery | Many products have broken option selects |

---

## Cross-site gotchas

### Counting buttons
`vibium count "button"` has a JSON parse error on some sites. Use eval:
```bash
vibium eval 'document.querySelectorAll("button").length'
```

### Clicking obscured buttons
When `vibium click` fails with "receivesEvents check failed — element is obscured" or "zero size", scroll first or bypass with eval:
```bash
vibium scroll into-view "button[title='Add to Cart']" && vibium sleep 500
vibium click @eN
# Still failing? Use eval:
vibium eval 'document.querySelector("button[title=\"Add to Cart\"]").click()'
```

### Disabled Add to Cart (broken required options)
On some sites (OpenCart), products have required option `<select>` elements that contain only a placeholder — no real values. This disables the Add to Cart button entirely. There's no user path to resolve it.

To detect: check if the button is truly disabled vs just obscured:
```bash
vibium eval 'document.querySelector("button[title=\"Add to Cart\"]").disabled'
# true = disabled (check for broken options)
# false = just obscured, use eval.click()
```
```bash
vibium eval 'JSON.stringify([...document.querySelectorAll("select option")].map(o => o.textContent.trim()))'
# If only ["--- Please Select ---", "--- Please Select ---"] — broken product, report as bug
```
**Resolution:** find a different in-stock product without required options.

### Country/state dropdowns — OpenCart (numeric IDs)
OpenCart dropdowns use internal numeric IDs. ISO codes and display names both fail silently. Always inspect option values first:
```bash
vibium eval 'JSON.stringify([...document.querySelector("#input-payment-country").options].filter(o=>o.text.includes("United States")).map(o=>({v:o.value,t:o.text})))'
# → [{"v":"223","t":"United States"}]
vibium select "#input-payment-country" "223"
vibium wait load && vibium sleep 1000  # triggers zone dropdown reload
vibium eval 'JSON.stringify([...document.querySelector("#input-payment-zone").options].filter(o=>o.text.includes("Texas")).map(o=>({v:o.value,t:o.text})))'
# → [{"v":"3669","t":"Texas"}]
vibium select "#input-payment-zone" "3669"
```

### Country/state dropdowns — Shopify (ISO codes)
Shopify uses ISO codes, not display names. `vibium select @e7 "United States"` silently fails:
```bash
vibium select @e7 "US"    # country
vibium select @e14 "TX"   # state
```
Selecting country triggers a full page reload and clears address fields — always refill after `vibium wait load`.

### Checkbox reset after zone reload (OpenCart)
Selecting a country or state on OpenCart triggers a partial page reload that silently unchecks all checkboxes (Terms & Conditions, Privacy Policy, newsletter). Always re-check after any zone selection, as the final step before submitting:
```bash
vibium eval 'document.querySelector("[name=agree]").click()'
```

### Obscured checkboxes
Styled checkboxes are often visually covered by `<label>` elements and fail with "element is obscured". Use eval to click the underlying input:
```bash
vibium eval 'document.querySelector("[name=agree]").click()'
vibium eval 'document.querySelector("[name=account_agree]").click()'
```

### Guest checkout (OpenCart)
The guest checkout radio is also obscured. Use eval:
```bash
vibium eval 'document.querySelector("[name=account][value=guest]").click()'
```

### One-page checkout (OpenCart)
OpenCart puts all checkout fields on a single form: personal details, billing address, shipping method, payment method, and terms. Fill top-to-bottom:
1. Personal details (first name, last name, email, phone)
2. Billing address (address, city, postcode)
3. Country → wait for zone reload → state
4. Re-check Terms & Conditions
5. Submit via `vibium eval 'document.querySelector("#button-save").click()'`

The form then navigates to a "Confirm Order" review page before final submission.

### Hidden quantity inputs (Shopify)
Shopify carts render duplicate zero-size hidden quantity inputs alongside visible ones. Filter for visible:
```bash
vibium eval 'const inputs = [...document.querySelectorAll("input[name=\"updates[]\"]")].filter(el => el.offsetWidth > 0); inputs[0].value = "2"; inputs[0].dispatchEvent(new Event("change")); "done"'
vibium find "input[name='update']" && vibium click @e1 && vibium wait load
```

### PCI payment iframes (Shopify)
Shopify payment fields are in cross-origin iframes — `vibium fill` is blocked by the browser. Get the bounding rect, click the center, then type one character at a time:
```bash
vibium eval 'JSON.stringify(document.querySelector("[id^=card-fields-number]")?.getBoundingClientRect())'
vibium mouse click <center-x> <center-y>
vibium keys "1"   # type one digit at a time — multi-char strings fail inside iframes
```
Iframe IDs change per session. Discover them fresh each run:
```bash
vibium eval 'JSON.stringify([...document.querySelectorAll("iframe")].map((f,i) => ({i, id: f.id, src: f.src?.substring(0,60)})))'
```

### Cart state persistence
- **var.parts**: In-memory — cart is lost on any direct URL navigation to `/cart`. Always use the header cart badge.
- **Shopify**: Server-side session — direct `/cart` URL is safe regardless of navigation path.
- **saucedemo.com**: Server-side session — direct `/cart.html` URL is safe.
- **OpenCart (lambdatest)**: Server-side session — direct cart URL is safe.

### Daemon recovery
On long sessions, the vibium daemon can drop with "broken pipe" or "i/o timeout". Restart and re-navigate:
```bash
vibium daemon stop && sleep 2 && vibium daemon start && sleep 2
vibium go <url> && vibium wait load
# Re-login if needed
```

---

## Site: var.parts

**URL:** https://var.parts/  
**Platform:** Custom React  
**Products:** 12 items — SWAG, ACCESSORIES, HEAD UNITS, CHASSIS, WIRING, HARDWARE  
**Cart:** In-memory — never `vibium go https://var.parts/cart` after adding items

### Test flows

#### 1. Navigation
```bash
vibium go https://var.parts/ && vibium map
# Verify: @e2 [a] "Shop", @e3 [a] "About" present
vibium click @e3 && vibium wait load && vibium url
# Expected: https://var.parts/about

vibium back && vibium wait load && vibium map
vibium click @e2 && vibium wait load && vibium url
# Expected: https://var.parts/
```

#### 2. Product listing
```bash
vibium go https://var.parts/ && vibium map
# Verify: 12 "Add to Cart" buttons present
vibium eval 'document.querySelectorAll("button").length'
# Verify: count >= 12
```

#### 3. Product detail
```bash
vibium go https://var.parts/
vibium find text "Vibium Battery Pack" && vibium click @e1 && vibium wait load
# Verify: URL = https://var.parts/product/12
vibium text
# Verify: name, price, "Install Time", "Compatibility", "Warranty", Add to Cart button
```

#### 4. Add to cart from listing
```bash
vibium go https://var.parts/ && vibium map
vibium click @e8   # first "Add to Cart"
vibium wait text "Added to cart"
vibium diff map
# Verify: button changed to "In Cart", cart badge count = 1
```

#### 5. Add to cart from product page
```bash
vibium go https://var.parts/product/12 && vibium map
vibium click @e8   # "Add to Cart" on product page
vibium wait text "Added to cart"
vibium diff map
# Verify: button changed to "In Cart"
```

#### 6. Cart management
```bash
# Add two items first, then navigate via cart badge — never direct URL
vibium map
vibium click @e6   # cart badge (e.g. "2")
vibium wait load
# Verify: URL = https://var.parts/cart, item count matches

vibium map
vibium click @e7   # "+" button for first item
vibium text
# Verify: quantity and subtotal updated

vibium click @e6   # "-" button
vibium text
# Verify: quantity decremented

vibium click @e6   # decrement to 0 removes item
vibium text
# Verify: item removed

vibium map
vibium click @e17  # "Clear Cart"
vibium text
# Verify: "Your cart is empty"
```

#### 7. Empty cart state
```bash
vibium go https://var.parts/cart
# Verify: "Your cart is empty"
vibium go https://var.parts/checkout
# Verify: "Nothing to check out"
```

#### 8. Checkout form validation
```bash
# Add item, navigate via cart badge to cart, proceed to checkout, submit empty
vibium find text "Proceed to Payment" && vibium click @e1
vibium text
# Verify: "Missing address" and "Unit designation and service bay are required."
```

#### 9. Full checkout — Lunar shipping (happy path)
```bash
vibium go https://var.parts/ && vibium map
vibium click @e8 && vibium wait text "Added to cart"
vibium map
vibium click @e6   # cart badge
vibium wait load
vibium map
vibium click @e11  # "Proceed to Checkout"
vibium wait load

vibium map
vibium fill @e7 "VAR-402"    # Unit Designation
vibium fill @e8 "Bay 14-C"   # Service Bay

vibium click @e14  # "Proceed to Payment"
vibium wait load && vibium text
# Verify: "Complete Payment", "Pay Here" and "Other Device" tabs
# Verify: "Ship to: VAR-402", "Bay: Bay 14-C"

vibium map
vibium click @e10  # "Pay Now"
vibium wait text "Payment Received!"
# Verify: "parts will be dispatched to Bay 14-C, VAR-402"

vibium map
vibium click @e7   # "Back to Shop"
vibium wait load && vibium url
# Expected: https://var.parts/, cart badge gone
```

#### 10. Checkout — Interplanetary shipping
```bash
# After filling form, before clicking Proceed to Payment:
vibium map
vibium click @e13   # Interplanetary Express radio
vibium text
# Verify: Shipping = ⚡149.99, Total = item price + 149.99
```

#### 11. Payment — Other Device tab
```bash
vibium map
vibium click @e8   # "Other Device" tab
vibium diff map
# Verify: "Copy payment link" button, QR prompt, "Waiting for payment confirmation..."

vibium click @e11  # "Copy payment link"
vibium text
# Verify: "Link copied to clipboard"
```

#### 12. Back to Details / Back to Cart
```bash
vibium map
vibium click @e12  # "Back to Details"
vibium diff map
# Verify: checkout form inputs reappear

vibium map
vibium click @e6   # "Back to Cart"
vibium wait load && vibium url
# Expected: https://var.parts/cart, items still present
```

#### Quick smoke test
```bash
vibium go https://var.parts/ && vibium map
vibium click @e3 && vibium wait load && vibium url
# Expected: https://var.parts/about

vibium go https://var.parts/ && vibium map
vibium click @e8 && vibium wait text "Added to cart"
vibium click @e11 && vibium wait text "Added to cart"

vibium map
vibium click @e7   # cart badge "2"
vibium wait load && vibium text
# Expected: "2 items in your cart", total = ⚡239.98

vibium map
vibium click @e11 && vibium wait load
vibium map
vibium fill @e7 "VAR-999" && vibium fill @e8 "Bay 99-Z"
vibium click @e14 && vibium wait load
vibium map
vibium click @e10 && vibium wait text "Payment Received!"
vibium map
vibium click @e7 && vibium wait load && vibium url
# Expected: https://var.parts/
```

---

## Site: sauce-demo.myshopify.com

**URL:** https://sauce-demo.myshopify.com/  
**Catalog:** https://sauce-demo.myshopify.com/collections/all  
**Platform:** Shopify  
**Products:** 7 total, 2 sold out (Brown Shades, White sandals), currency GBP  
**Cart:** Server-side session — direct `/cart` URL is safe

### Test flows

#### 1. Navigation
```bash
vibium go https://sauce-demo.myshopify.com/ && vibium map
# Verify: Home, Catalog, Blog, About Us, Wish list, Refer a friend nav links present
vibium click @e11 && vibium wait load && vibium url
# Expected: /collections/all

vibium map
vibium click @e12 && vibium wait load && vibium url
# Expected: /blogs/news

vibium map
vibium click @e13 && vibium wait load && vibium url
# Expected: /pages/about-us
```

#### 2. Product listing
```bash
vibium go https://sauce-demo.myshopify.com/collections/all && vibium text
# Verify: Black heels £45, Bronze sandals £39.99, Brown Shades £20 (SOLD OUT),
#         Grey jacket £55, Noir jacket £60, Striped top £50, White sandals £25 (SOLD OUT)
```

#### 3. Product detail
```bash
vibium go https://sauce-demo.myshopify.com/products/grey-jacket && vibium wait load
vibium text
# Verify: name "Grey jacket", price "£55.00", Add to Cart present
```

#### 4. Add to cart
```bash
vibium go https://sauce-demo.myshopify.com/products/grey-jacket && vibium map
vibium find "input[name='add']" && vibium click @e1 && vibium wait load
# Verify: cart count incremented to 1
```

#### 5. Multi-variant add to cart (Noir jacket)
```bash
vibium go https://sauce-demo.myshopify.com/products/noir-jacket && vibium map
# @e23 = size select (S/M/L), @e24 = color select (Blue/Red)
# The submit button comes AFTER the selects — it is @e25, not @e24
vibium click @e25 && vibium wait load
# Verify: cart count incremented
```

#### 6. Cart page
```bash
vibium go https://sauce-demo.myshopify.com/cart && vibium text
# Verify: items listed with correct prices and total
```

#### 7. Quantity update
```bash
# Shopify renders hidden duplicate qty inputs — filter for visible ones
vibium eval 'const inputs = [...document.querySelectorAll("input[name=\"updates[]\"]")].filter(el => el.offsetWidth > 0); inputs[0].value = "2"; inputs[0].dispatchEvent(new Event("change")); "done"'
vibium find "input[name='update']" && vibium click @e1 && vibium wait load
vibium text
# Verify: quantity = 2, subtotal doubled, total updated
```

#### 8. Remove item
```bash
vibium map
vibium click @e25   # first item remove link ("x")
vibium wait load && vibium text
# Verify: item removed, total updated
```

#### 9. Checkout form validation
```bash
vibium find "input[name='checkout']" && vibium click @e1 && vibium wait load
vibium find "button[type='submit']" && vibium click @e1 && vibium sleep 2000
vibium text
# Verify: "Enter an address", "Enter a city", "Enter a ZIP / postal code",
#         "Enter a card number", "Enter a valid expiration date"
```

#### 10. Full checkout (happy path)
```bash
# 1. Add item and navigate to checkout
vibium go https://sauce-demo.myshopify.com/products/grey-jacket && vibium map
vibium find "input[name='add']" && vibium click @e1 && vibium wait load
vibium go https://sauce-demo.myshopify.com/cart
vibium find "input[name='checkout']" && vibium click @e1 && vibium wait load

# 2. Fill delivery — country MUST use ISO code, not display name
vibium fill @e5 "test@example.com"
vibium select @e7 "US" && vibium wait load   # triggers reload, clears address fields
vibium select @e14 "TX"
vibium fill @e9 "Tester"
vibium fill @e11 "123 Test Street"
vibium fill @e13 "Austin"
vibium fill @e15 "78701"
vibium fill @e16 "5125550100"

# 3. Wait for shipping to load (requires complete address)
vibium sleep 3000 && vibium text
# Verify: "International Shipping" option appears with £20

# 4. Fill payment via PCI iframes (cross-origin — vibium fill is blocked)
vibium eval 'JSON.stringify({num: document.querySelector("[id^=card-fields-number]")?.getBoundingClientRect(), exp: document.querySelector("[id^=card-fields-expiry]")?.getBoundingClientRect(), cvv: document.querySelector("[id^=card-fields-verification_value]")?.getBoundingClientRect(), name: document.querySelector("[id^=card-fields-name]")?.getBoundingClientRect()})'
# Card: click center of num rect → vibium keys "1"
# Expiry: click center of exp rect → vibium keys "1","2","3","0"  (= 12/30)
# CVV: click center of cvv rect → vibium keys "1","2","3"
# Name: click center of name rect → vibium keys "T","e","s","t","e","r"

# 5. Submit
vibium find "button[type='submit']" && vibium click @e1 && vibium sleep 4000
vibium url
# Expected: URL contains /thank-you
vibium text
# Verify: "Thank you!", "Your order is confirmed", order number, summary
```

---

## Site: saucedemo.com

**URL:** https://www.saucedemo.com  
**Platform:** Custom React, login-gated demo  
**Login:** `standard_user` / `secret_sauce` (shown on login page)  
**Products:** 6, all in stock, USD  
**Cart:** Server-side session — direct `/cart.html` URL is safe

**Known bugs:**
- Empty cart checkout not blocked — proceeds to checkout form with no warning
- "About" burger menu link navigates to an external page that hangs (recover with `vibium go`)

**Cart limitation:** No quantity +/- controls — add/remove only.

### Test flows

#### 1. Login
```bash
vibium go https://www.saucedemo.com && vibium map
vibium fill @e1 "standard_user" && vibium fill @e2 "secret_sauce" && vibium click @e3
vibium wait load && vibium url
# Expected: https://www.saucedemo.com/inventory.html
```

#### 2. Navigation
```bash
vibium click "#react-burger-menu-btn" && vibium sleep 500
vibium map --selector ".bm-menu"
# Verify: All Items, About, Logout, Reset App State present
vibium click "#react-burger-cross-btn" && vibium sleep 600
# Note: "About" goes to external site that may hang — skip or recover with vibium go
```

#### 3. Product listing
```bash
vibium text ".inventory_list"
# Verify: 6 products with names and prices
vibium eval 'document.querySelectorAll("button").length'
# Verify: 8 buttons (6 Add to cart + 2 header)
```

#### 4. Product detail
```bash
vibium find text "Sauce Labs Backpack" && vibium click @e1
vibium wait load && vibium url
# Expected: /inventory-item.html?id=4
vibium text
# Verify: name, price ($29.99), description, Add to cart button
```

#### 5. Add to cart
```bash
vibium find text "Add to cart" && vibium click @e1
vibium diff map
# Verify: button changed to "Remove"
vibium eval 'document.querySelector(".shopping_cart_badge")?.textContent'
# Expected: "1"
```

#### 6. Cart management
```bash
vibium click ".shopping_cart_link" && vibium wait load && vibium url
# Expected: /cart.html
vibium text
# Verify: item present with correct price, Remove button visible
# Note: no quantity controls

vibium find text "Remove" && vibium click @e1
vibium diff map
# Verify: item gone
vibium eval 'document.querySelector(".shopping_cart_badge")?.textContent || "empty"'
# Expected: "empty"
```

#### 7. Empty cart — checkout not blocked (known bug)
```bash
vibium find text "Checkout" && vibium click @e1 && vibium wait load && vibium url
# Actual:   /checkout-step-one.html  (BUG — should be blocked)
# Expected: empty-cart warning or redirect
```

#### 8. Checkout form validation
```bash
# From /checkout-step-one.html:
vibium click "#continue" && vibium text
# Verify: "Error: First Name is required"
vibium fill "#first-name" "Test" && vibium click "#continue" && vibium text
# Verify: "Error: Last Name is required"
vibium fill "#last-name" "User" && vibium click "#continue" && vibium text
# Verify: "Error: Postal Code is required"
```

#### 9. Full checkout (happy path)
```bash
# 1. Add item
vibium go https://www.saucedemo.com/inventory.html && vibium wait load
vibium find text "Add to cart" && vibium click @e1

# 2. Checkout
vibium click ".shopping_cart_link" && vibium wait load
vibium find text "Checkout" && vibium click @e1 && vibium wait load
# Expected: /checkout-step-one.html

# 3. Fill info
vibium fill "#first-name" "Test"
vibium fill "#last-name" "User"
vibium fill "#postal-code" "12345"
vibium click "#continue" && vibium wait load
# Expected: /checkout-step-two.html

# 4. Review
vibium text
# Verify: item + price, "SauceCard #31337", "Free Pony Express Delivery!", total with tax

# 5. Confirm
vibium find text "Finish" && vibium click @e1 && vibium wait load && vibium url
# Expected: /checkout-complete.html
vibium text
# Verify: "Checkout: Complete!", "Thank you for your order!"
vibium eval 'document.querySelector(".shopping_cart_badge")?.textContent || "empty"'
# Expected: "empty"
```

#### Quick smoke test
```bash
vibium go https://www.saucedemo.com && vibium map
vibium fill @e1 "standard_user" && vibium fill @e2 "secret_sauce" && vibium click @e3
vibium wait load && vibium url
# Expected: /inventory.html

vibium find text "Add to cart" && vibium click @e1
vibium eval 'document.querySelector(".shopping_cart_badge")?.textContent'
# Expected: "1"

vibium click ".shopping_cart_link" && vibium wait load
vibium find text "Checkout" && vibium click @e1 && vibium wait load
vibium fill "#first-name" "Test" && vibium fill "#last-name" "User" && vibium fill "#postal-code" "12345"
vibium click "#continue" && vibium wait load
vibium find text "Finish" && vibium click @e1 && vibium wait load && vibium url
# Expected: /checkout-complete.html
```

---

## Site: ecommerce-playground.lambdatest.io

**URL:** https://ecommerce-playground.lambdatest.io/  
**Platform:** OpenCart  
**Products:** 75+ per category across Desktops, Laptops, Cameras, Phones, Software, MP3 Players, and more  
**Cart:** Server-side session — direct cart URL is safe  
**Reliable test product:** HP LP3065 (`product_id=47`) — in stock, $122.00, no required options

**Known bugs:**
- Many products have required option selects with only a placeholder option — Add to Cart is permanently disabled on those products with no path to resolve it
- `/checkout/success` page is accessible directly without placing an order — shows full confirmation
- Country/state zone reload silently unchecks Terms & Conditions checkbox
- Post-order "Continue" button loops to the same success URL instead of going home

### Test flows

#### 1. Navigation
```bash
vibium go https://ecommerce-playground.lambdatest.io/ && vibium map
vibium map --selector "#widget-navbar-217834"
# Verify: Home, Special, Blog, Mega Menu, AddOns, My account present

vibium click @e2 && vibium wait load && vibium url
# Expected: ?route=product/special

vibium back && vibium wait load
vibium map --selector "#widget-navbar-217834"
vibium click @e3 && vibium wait load && vibium url
# Expected: ?route=extension/maza/blog/home

vibium back && vibium wait load
vibium map --selector "#widget-navbar-217834"
vibium click @e6 && vibium wait load && vibium url
# Expected: ?route=account/login
```

#### 2. Product listing
```bash
vibium go "https://ecommerce-playground.lambdatest.io/index.php?route=product/category&path=20" && vibium wait load
vibium text
# Verify: product names and prices visible (HTC Touch HD $146, Palm Treo Pro $337.99, etc.)
vibium eval 'document.querySelectorAll("button").length'
# Verify: many buttons (Add to Cart, Wish List, Quick view, Compare per product)
```

#### 3. Product detail
```bash
vibium go "https://ecommerce-playground.lambdatest.io/index.php?route=product/product&product_id=47" && vibium wait load
vibium eval 'document.querySelector("h1").textContent.trim()'
# Expected: "HP LP3065"
vibium eval 'document.querySelector(".price").textContent.trim()'
# Expected includes: "$122.00"
vibium find text "In Stock"
# Verify: availability shown

# Check Add to Cart is enabled (no broken options on this product)
vibium eval 'document.querySelectorAll("select").length'
# Expected: 0 (no required option selects)
vibium eval 'document.querySelector("#product-product button[title=\"Add to Cart\"]").disabled'
# Expected: false
```

#### 4. Add to cart
```bash
vibium go "https://ecommerce-playground.lambdatest.io/index.php?route=product/product&product_id=47" && vibium wait load

# The Add to Cart button may be initially obscured — scroll into view, then use eval.click()
vibium scroll into-view "#product-product button[title=\"Add to Cart\"]" && vibium sleep 500
vibium eval 'document.querySelector("#entry_216842 button").click()'
vibium sleep 1500
# Verify: toast "Success: You have added HP LP3065 to your shopping cart!"
# Verify: cart badge shows "1 item(s) - $122.00"
```

#### 5. Cart management
```bash
vibium go "https://ecommerce-playground.lambdatest.io/index.php?route=checkout/cart" && vibium wait load
vibium text
# Verify: HP LP3065 listed, $122.00, quantity = 1

# Update quantity to 2
vibium eval 'document.querySelector("input[name*=quantity]").value = "2"; document.querySelector("input[name*=quantity]").dispatchEvent(new Event("change")); "done"'
vibium find title "Update" && vibium click @e1 && vibium wait load
vibium text
# Verify: quantity = 2, total = $244.00

# Remove item
vibium find title "Remove" && vibium click @e1 && vibium wait load
vibium text "#content"
# Verify: "Your shopping cart is empty!"
```

#### 6. Empty cart state
```bash
# With empty cart, attempt to go to checkout
vibium go "https://ecommerce-playground.lambdatest.io/index.php?route=checkout/checkout" && vibium wait load && vibium url
# Expected: redirected back to ?route=checkout/cart (checkout correctly blocked)

# Also verify success page is accessible without an order (known bug)
vibium go "https://ecommerce-playground.lambdatest.io/index.php?route=checkout/success" && vibium wait load
vibium text "#content"
# Actual: "Your order has been successfully processed!" (BUG — no order was placed)
```

#### 7. Checkout form validation
```bash
# Add item first, then go to checkout
vibium go "https://ecommerce-playground.lambdatest.io/index.php?route=product/product&product_id=47" && vibium wait load
vibium eval 'document.querySelector("#entry_216842 button").click()' && vibium sleep 1000
vibium go "https://ecommerce-playground.lambdatest.io/index.php?route=checkout/checkout" && vibium wait load

# Submit with empty form
vibium eval 'document.querySelector("#button-save").click()' && vibium sleep 1500
# Verify: "Warning: You must agree to the Terms & Conditions!" banner
# Verify: inline errors on First Name, Last Name, E-Mail, Telephone, Address 1, Post Code
```

#### 8. Full checkout (happy path)
```bash
# 1. Add item
vibium go "https://ecommerce-playground.lambdatest.io/index.php?route=product/product&product_id=47" && vibium wait load
vibium eval 'document.querySelector("#entry_216842 button").click()' && vibium sleep 1000

# 2. Go to checkout
vibium go "https://ecommerce-playground.lambdatest.io/index.php?route=checkout/checkout" && vibium wait load

# 3. Select guest checkout (radio is obscured — use eval)
vibium eval 'document.querySelector("[name=account][value=guest]").click()' && vibium sleep 500

# 4. Fill personal details
vibium map --selector "#content" | grep placeholder  # confirm field refs
vibium fill "#input-payment-firstname" "Test"
vibium fill "#input-payment-lastname" "User"
vibium fill "#input-payment-email" "guest_test@example.com"
vibium fill "#input-payment-telephone" "12025550100"

# 5. Fill billing address
vibium fill "#input-payment-address-1" "123 Test Street"
vibium fill "#input-payment-city" "Austin"
vibium fill "#input-payment-postcode" "78701"

# 6. Select country (numeric ID — display names fail silently)
vibium select "#input-payment-country" "223" && vibium wait load && vibium sleep 1000
# Page reloads; state dropdown now shows US states

# 7. Select state (numeric ID)
vibium select "#input-payment-zone" "3669" && vibium sleep 500
# Triggers another reload; Terms checkbox gets unchecked

# 8. Re-check Terms (reset by zone reload) — checkbox is obscured, use eval
vibium eval 'document.querySelector("[name=agree]").click()'

# 9. Submit
vibium eval 'document.querySelector("#button-save").click()' && vibium sleep 5000
vibium url
# Expected: ?route=extension/maza/checkout/confirm

# 10. Review confirm page
vibium text "#content"
# Verify: product name, price, billing address, shipping method (Flat Shipping Rate)

# 11. Confirm order
vibium find text "Confirm Order" --selector "button" 2>/dev/null || vibium map --selector "#content" | grep "button"
vibium click @e2   # "Confirm Order" button
vibium wait load && vibium url
# Expected: ?route=checkout/success
vibium text "#content"
# Verify: "Your order has been successfully processed!"

# 12. Verify cart cleared
vibium go "https://ecommerce-playground.lambdatest.io/" && vibium wait load
vibium find text "item(s)"
# Expected: "0   0 item(s) - $0.00"
```

#### Quick smoke test
```bash
# Add item
vibium go "https://ecommerce-playground.lambdatest.io/index.php?route=product/product&product_id=47" && vibium wait load
vibium eval 'document.querySelector("#entry_216842 button").click()' && vibium sleep 1500
# Verify: toast with "HP LP3065"

# Check cart
vibium go "https://ecommerce-playground.lambdatest.io/index.php?route=checkout/cart" && vibium wait load
vibium text
# Verify: HP LP3065, $122.00

# Remove item
vibium find title "Remove" && vibium click @e1 && vibium wait load
vibium text "#content"
# Verify: "Your shopping cart is empty!"

# Empty cart checkout redirect
vibium go "https://ecommerce-playground.lambdatest.io/index.php?route=checkout/checkout" && vibium wait load && vibium url
# Expected: redirected to ?route=checkout/cart
```
