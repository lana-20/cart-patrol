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
/cart-patrol https://sauce-demo.myshopify.com/ checkout
/cart-patrol smoke
```

**Flow names:** `navigation`, `product`, `cart`, `checkout`, `smoke`, `all`

## Tested sites

| Site | Type | Notes |
|------|------|-------|
| [var.parts](https://var.parts/) | Custom React | In-memory cart, custom payment UI |
| [sauce-demo.myshopify.com](https://sauce-demo.myshopify.com/) | Shopify | Server-side cart, PCI iframe payment |
| [saucedemo.com](https://www.saucedemo.com) | Custom React (demo) | Login-gated, fake payment, no qty controls in cart |

---

## Cross-site gotchas

### Counting buttons
`vibium count "button"` has a JSON parse bug on some sites. Use:
```bash
vibium eval 'document.querySelectorAll("button").length'
```

### Hidden quantity inputs (Shopify)
Shopify carts render duplicate zero-size hidden quantity inputs alongside visible ones. Target visible inputs via eval:
```bash
vibium eval 'const inputs = [...document.querySelectorAll("input[name=\"updates[]\"]")].filter(el => el.offsetWidth > 0); inputs[0].value = "2"; inputs[0].dispatchEvent(new Event("change")); "done"'
```

### PCI payment iframes (Shopify)
Shopify payment fields live in cross-origin iframes — `vibium fill` is blocked by the browser. Workaround: get iframe bounding rect, click by coordinates, type one character at a time:
```bash
vibium eval 'JSON.stringify(document.querySelector("#card-fields-number-<id>")?.getBoundingClientRect())'
vibium mouse click <center-x> <center-y>
vibium keys "4"   # type digits one by one — multi-char strings fail
```
Iframe IDs contain random suffixes; fetch them fresh each session via:
```bash
vibium eval 'JSON.stringify([...document.querySelectorAll("iframe")].map((f,i) => ({i, name: f.name, src: f.src?.substring(0,60)})))'
```

### Country/state ISO codes (Shopify checkout)
Shopify's country/state dropdowns use ISO codes internally. `vibium select @e7 "United States"` silently fails. Use the ISO value:
```bash
vibium select @e7 "US"    # country
vibium select @e14 "TX"   # state
```
Selecting country triggers a page reload and clears address fields — always refill address, city, and ZIP after `vibium wait load`.

### Cart state persistence
- **var.parts**: in-memory — never navigate directly to `/cart` after adding items or cart will appear empty. Use UI (header cart badge).
- **Shopify**: server-side session — direct `/cart` navigation is safe.

---

## Site: var.parts

**URL:** https://var.parts/

**Products:** 12 items across SWAG, ACCESSORIES, HEAD UNITS, CHASSIS, WIRING, HARDWARE

### Test flows

#### 1. Navigation
```bash
vibium go https://var.parts/ && vibium map
# Verify: @e2 [a] "Shop", @e3 [a] "About" present
vibium click @e3 && vibium wait load
# Verify: URL = https://var.parts/about, page contains "About VAR Parts"
vibium click @e2 && vibium wait load
# Verify: URL = https://var.parts/
```

#### 2. Product listing
```bash
vibium go https://var.parts/ && vibium map
# Verify: 12 "Add to Cart" buttons present
vibium eval 'document.querySelectorAll("button").length'
# Verify: count >= 12
```

#### 3. Product detail page
```bash
vibium go https://var.parts/
vibium map
vibium click @e7   # "Vibium Battery Pack" title link
vibium wait load
# Verify: URL = https://var.parts/product/12
# Verify: page contains "Install Time", "Compatibility", "Warranty"
# Verify: "Add to Cart" button present, "Compatible Components" section present
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
# Start: add two items (flow 4), then navigate via cart badge — never via direct URL
vibium map
vibium click @e6   # cart badge (e.g. "2")
vibium wait load
# Verify: URL = https://var.parts/cart, item count in heading matches

vibium map
vibium click @e7   # "+" button for first item
vibium text
# Verify: quantity and subtotal updated

vibium click @e6   # "-" button for first item
vibium text
# Verify: quantity decremented

vibium click @e6   # decrement to 0 removes item
vibium text
# Verify: item removed from cart

vibium map
vibium click @e17  # "Clear Cart" button
vibium text
# Verify: "Your cart is empty"
```

#### 7. Empty cart state
```bash
vibium go https://var.parts/cart
# Verify: "Your cart is empty" (direct navigation with no session)
vibium go https://var.parts/checkout
# Verify: "Nothing to check out"
```

#### 8. Checkout form validation
```bash
# Add item, navigate via UI to checkout, then submit empty:
vibium find text "Proceed to Payment"
vibium click @e1
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
vibium fill @e7 "VAR-402"    # Unit Designation (required)
vibium fill @e8 "Bay 14-C"   # Service Bay (required)

vibium click @e14  # "Proceed to Payment"
vibium wait load && vibium text
# Verify: "Complete Payment", "Pay Here" tab, "Other Device" tab
# Verify: "Ship to: VAR-402", "Bay: Bay 14-C"

vibium map
vibium click @e10  # "Pay Now"
vibium wait text "Payment Received!"
# Verify: "parts will be dispatched to Bay 14-C, VAR-402"

vibium map
vibium click @e7   # "Back to Shop"
vibium wait load
# Verify: URL = https://var.parts/, cart badge gone
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
# Verify: "Copy payment link" button appears
# Verify: "Scan this code with another device's payment terminal"
# Verify: "Waiting for payment confirmation..."

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
vibium wait load
# Verify: URL = https://var.parts/cart, items still in cart
```

#### Quick smoke test
```bash
vibium go https://var.parts/ && vibium map
vibium click @e3 && vibium wait load && vibium url
# Expected: https://var.parts/about

vibium go https://var.parts/product/12 && vibium wait load && vibium url
# Expected: https://var.parts/product/12

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

**Products:** 7 total (2 sold out — Brown Shades, White sandals), currency GBP

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
# Verify: URL = /products/grey-jacket
vibium map
# Verify: variant select (@e23), Add to Cart submit button (@e24) present
vibium text
# Verify: name "Grey jacket", price "£55.00"
```

#### 4. Add to cart
```bash
vibium go https://sauce-demo.myshopify.com/products/grey-jacket && vibium map
vibium find "input[name='add']"
vibium click @e1 && vibium wait load
vibium text "a#minicart, [class*='cart']"
# Verify: My Cart count incremented to 1
```

#### 5. Multi-variant add to cart (Noir jacket)
```bash
vibium go https://sauce-demo.myshopify.com/products/noir-jacket && vibium map
# @e23 = size select (S/M/L), @e24 = color select (Blue/Red), @e25 = Add to Cart
# Variant selects appear BEFORE the submit button — click @e25, not @e24
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
# Quantity inputs may include hidden zero-size clones — use eval to find visible ones:
vibium eval 'const inputs = [...document.querySelectorAll("input[name=\"updates[]\"]")].filter(el => el.offsetWidth > 0); inputs[0].value = "2"; inputs[0].dispatchEvent(new Event("change")); "done"'
vibium find "input[name='update']" && vibium click @e1 && vibium wait load
vibium text
# Verify: quantity = 2, subtotal doubled, order total updated
```

#### 8. Remove item
```bash
vibium map
# Find "x" remove links
vibium click @e25   # first item remove
vibium wait load && vibium text
# Verify: item removed, cart count decremented, total updated
```

#### 9. Checkout form validation
```bash
# Add item, go to cart, click checkout:
vibium find "input[name='checkout']" && vibium click @e1 && vibium wait load
# Click Pay now without filling anything:
vibium find "button[type='submit']" && vibium click @e1 && vibium sleep 2000
vibium text
# Verify: "Enter an address", "Enter a city", "Enter a ZIP / postal code",
#         "Enter a card number", "Enter a valid expiration date"
```

#### 10. Full checkout (happy path)
```bash
# 1. Add item and go to checkout
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

# 3. Wait for shipping methods to load (requires complete address)
vibium sleep 3000 && vibium text
# Verify: "International Shipping" option appears with price

# 4. Fill payment fields — PCI iframes require mouse clicks + single-char key presses
# Get iframe positions:
vibium eval 'JSON.stringify({num: document.querySelector("[id^=card-fields-number]")?.getBoundingClientRect(), exp: document.querySelector("[id^=card-fields-expiry]")?.getBoundingClientRect(), cvv: document.querySelector("[id^=card-fields-verification_value]")?.getBoundingClientRect(), name: document.querySelector("[id^=card-fields-name]")?.getBoundingClientRect()})'
# Then click each iframe center and type one digit at a time:
# Card: mouse click <num-center-x> <num-center-y> → keys "1"
# Expiry: mouse click <exp-center-x> <exp-center-y> → keys "1","2","3","0"  (= 12/30)
# CVV: mouse click <cvv-center-x> <cvv-center-y> → keys "1","2","3"
# Name: mouse click <name-center-x> <name-center-y> → keys "T","e","s","t","e","r"

# 5. Submit
vibium find "button[type='submit']" && vibium click @e1 && vibium sleep 4000
vibium url
# Verify: URL contains /thank-you
vibium text
# Verify: "Thank you!", "Your order is confirmed", confirmation number, order summary
```

---

## Site: saucedemo.com

**URL:** https://www.saucedemo.com  
**Type:** Login-gated demo store (custom React, no real payment)

**Login credentials** (shown on login page):
- Username: `standard_user` (use this for all flows)
- Password: `secret_sauce`
- Other users: `locked_out_user`, `problem_user`, `performance_glitch_user`, `error_user`, `visual_user`

**Products:** 6 items, all available — Sauce Labs Backpack ($29.99), Bike Light ($9.99), Bolt T-Shirt ($15.99), Fleece Jacket ($49.99), Onesie ($7.99), Test.allTheThings() T-Shirt Red ($15.99)

**Cart:** Server-side session — direct URL navigation to `/cart` is safe.

**Payment:** No real payment form. Uses fake "SauceCard #31337" automatically. No PCI iframes.

**Known bugs:**
- Checkout with empty cart is not blocked — proceeds to checkout form with no warning
- "About" link in burger menu navigates to an external site that hangs (long load / timeout)

**Cart limitation:** No quantity +/− controls. Cart only supports add/remove. Quantity shown as label, not editable.

### Test flows

#### 1. Login
```bash
vibium go https://www.saucedemo.com && vibium map
vibium fill @e1 "standard_user" && vibium fill @e2 "secret_sauce" && vibium click @e3
vibium wait load && vibium url
# Verify: URL = https://www.saucedemo.com/inventory.html
```

#### 2. Navigation
```bash
# Burger menu — open, check links, close before next interaction
vibium click "#react-burger-menu-btn" && vibium sleep 500
vibium map --selector ".bm-menu"
# Verify: All Items, About, Logout, Reset App State present
vibium click "#react-burger-cross-btn" && vibium sleep 600
# Note: "About" navigates to external site that may hang — skip or use vibium go to recover
```

#### 3. Product listing
```bash
vibium go https://www.saucedemo.com/inventory.html && vibium wait load
vibium text ".inventory_list"
# Verify: 6 products with names and prices visible
vibium eval 'document.querySelectorAll("button").length'
# Verify: 8 buttons (6 Add to cart + 2 header)
```

#### 4. Product detail
```bash
vibium find text "Sauce Labs Backpack" && vibium click @e1
vibium wait load && vibium url
# Verify: URL = /inventory-item.html?id=4
vibium text
# Verify: product name, price ($29.99), description, "Add to cart" button
```

#### 5. Add to cart
```bash
vibium find text "Add to cart" && vibium click @e1
vibium diff map
# Verify: button changed to "Remove"
vibium eval 'document.querySelector(".shopping_cart_badge")?.textContent'
# Verify: "1"
```

#### 6. Cart management
```bash
vibium click ".shopping_cart_link" && vibium wait load && vibium url
# Verify: URL = /cart.html
vibium text
# Verify: added item present with correct price, Remove button present
# Note: no quantity controls — qty shown as static "1" label only

vibium find text "Remove" && vibium click @e1
vibium diff map
# Verify: item removed from cart list
vibium eval 'document.querySelector(".shopping_cart_badge")?.textContent || "empty"'
# Verify: "empty" (badge gone)
```

#### 7. Empty cart — checkout not blocked (known bug)
```bash
# With empty cart, click Checkout:
vibium find text "Checkout" && vibium click @e1 && vibium wait load && vibium url
# Actual: /checkout-step-one.html (proceeds without warning)
# Expected: empty-cart message or block
# STATUS: BUG — empty cart checkout is not blocked
```

#### 8. Checkout form validation
```bash
# From /checkout-step-one.html (with or without items):
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

# 2. Go to cart and checkout
vibium click ".shopping_cart_link" && vibium wait load
vibium find text "Checkout" && vibium click @e1 && vibium wait load
# Verify: URL = /checkout-step-one.html

# 3. Fill delivery info
vibium fill "#first-name" "Test"
vibium fill "#last-name" "User"
vibium fill "#postal-code" "12345"
vibium click "#continue" && vibium wait load
# Verify: URL = /checkout-step-two.html

# 4. Review order summary
vibium text
# Verify: item name + price, "SauceCard #31337", "Free Pony Express Delivery!",
#         Item total, Tax, Total

# 5. Finish
vibium find text "Finish" && vibium click @e1 && vibium wait load
vibium url
# Verify: URL = /checkout-complete.html
vibium text
# Verify: "Checkout: Complete!", "Thank you for your order!"
vibium eval 'document.querySelector(".shopping_cart_badge")?.textContent || "empty"'
# Verify: "empty" (cart cleared)
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
