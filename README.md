# cart-patrol

E2E test skill for https://var.parts/ — a test storefront for robot parts (Vibium Automation Robots). Uses vibium browser automation.

## Installation

**Via skills CLI (recommended):**
```bash
npx skills add lana-20/cart-patrol -g
```

**Manual:**
```bash
git clone https://github.com/lana-20/cart-patrol.git ~/.claude/skills/cart-patrol
```

## Usage

Once installed, invoke in Claude Code:
```
/cart-patrol
```

Or with a specific flow:
```
/cart-patrol smoke
/cart-patrol checkout
```

Requires [vibium](https://www.npmjs.com/package/vibium) to be installed:
```bash
npm install -g vibium
```

## What the site does

E-commerce shop with 12 products across categories (SWAG, ACCESSORIES, HEAD UNITS, CHASSIS, WIRING, HARDWARE). Full flow: browse → product detail → add to cart → cart management → checkout → payment.

**Important:** Cart state is in-memory only — lost on page reload or direct URL navigation. Always navigate via UI (cart badge/button), never `vibium go https://var.parts/cart` after adding items.

## Known site behavior

- Cart badge shows item count in header nav
- "Add to Cart" button changes to "In Cart" after adding
- Toast notification: "Added to cart [product name]"
- Copy payment link feedback: "Link copied to clipboard" (not "Copied")
- Decrement quantity to 0 removes item from cart
- Post-checkout: cart is cleared, confirmation shows "Payment Received!"
- Checkout form state is retained when going Back to Details from payment page

## Test flows

### 1. Navigation

```
vibium go https://var.parts/
vibium map
# Verify: @e2 [a] "Shop", @e3 [a] "About" present
vibium click @e3 && vibium wait load
# Verify: URL = https://var.parts/about, page contains "About VAR Parts"
vibium click @e2 && vibium wait load
# Verify: URL = https://var.parts/
```

### 2. Product listing

```
vibium go https://var.parts/ && vibium map
# Verify: 12 "Add to Cart" buttons present
vibium count "button"
# Verify: count >= 12
```

### 3. Product detail page

```
vibium go https://var.parts/
vibium map
vibium click @e7   # "Vibium Battery Pack" title link
vibium wait load
# Verify: URL = https://var.parts/product/12
# Verify: page contains "Install Time", "Compatibility", "Warranty"
# Verify: "Add to Cart" button present
# Verify: "Compatible Components" section present
```

### 4. Add to cart from listing

```
vibium go https://var.parts/
vibium map
vibium click @e8   # first "Add to Cart"
vibium wait text "Added to cart"
vibium diff map
# Verify: button changed to "In Cart"
# Verify: cart badge count = 1
```

### 5. Add to cart from product page

```
vibium go https://var.parts/product/12
vibium map
vibium click @e8   # "Add to Cart" on product page
vibium wait text "Added to cart"
vibium diff map
# Verify: button changed to "In Cart"
```

### 6. Cart management

```
# Start: add two items first (see flow 4), then navigate via cart badge
vibium map
vibium click @e6   # cart badge (e.g. "2")
vibium wait load
# Verify: URL = https://var.parts/cart
# Verify: item count in heading matches

# Quantity increment
vibium map
vibium click @e7   # "+" button for first item
vibium text
# Verify: quantity and subtotal updated

# Quantity decrement
vibium click @e6   # "-" button for first item
vibium text
# Verify: quantity decremented

# Decrement to 0 removes item
vibium click @e6
vibium text
# Verify: item removed from cart

# Clear Cart
vibium map
vibium click @e17  # "Clear Cart" button
vibium text
# Verify: "Your cart is empty"
```

### 7. Empty cart state

```
vibium go https://var.parts/cart
# Verify: "Your cart is empty" (direct navigation with no session)

vibium go https://var.parts/checkout
# Verify: "Nothing to check out"
```

### 8. Checkout form validation

```
# Add item, navigate via UI to checkout
# Then submit with empty required fields:
vibium find button "Proceed to Payment"
vibium click @e1
vibium text
# Verify: "Missing address" and "Unit designation and service bay are required."
```

### 9. Full checkout — Lunar shipping (happy path)

```
# 1. Add item
vibium go https://var.parts/
vibium map
vibium click @e8
vibium wait text "Added to cart"

# 2. Go to cart
vibium map
vibium click @e6   # cart badge
vibium wait load

# 3. Proceed to checkout
vibium map
vibium click @e11  # "Proceed to Checkout" button
vibium wait load

# 4. Fill form (Lunar shipping is default/FREE)
vibium map
vibium fill @e7 "VAR-402"    # Unit Designation (required)
vibium fill @e8 "Bay 14-C"   # Service Bay (required)
# @e9 Sector, @e10 Station are optional

# 5. Proceed to payment
vibium click @e14  # "Proceed to Payment"
vibium wait load
vibium text
# Verify: "Complete Payment", "Pay Here" tab, "Other Device" tab
# Verify: order total = ⚡89.99 (Battery Pack) or item price
# Verify: "Ship to: VAR-402" and "Bay: Bay 14-C"

# 6. Pay
vibium map
vibium click @e10  # "Pay Now"
vibium wait text "Payment Received!"
vibium text
# Verify: "Payment Received!" and "parts will be dispatched to Bay 14-C, VAR-402"

# 7. Return to shop
vibium map
vibium click @e7   # "Back to Shop"
vibium wait load
# Verify: URL = https://var.parts/, cart badge gone (cart cleared)
```

### 10. Checkout — Interplanetary shipping

```
# After filling form, before clicking Proceed to Payment:
vibium map
vibium click @e13   # Interplanetary Express radio
vibium text
# Verify: Shipping = ⚡149.99, Total = item price + 149.99
```

### 11. Payment — Other Device tab

```
# After reaching payment page:
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

### 12. Back to Details from payment

```
vibium map
vibium click @e12  # "Back to Details"
vibium diff map
# Verify: checkout form inputs reappear
```

### 13. Back to Cart from checkout

```
vibium map
vibium click @e6   # "Back to Cart"
vibium wait load
# Verify: URL = https://var.parts/cart, items still in cart
```

## Quick smoke test (all flows in sequence)

```bash
# Navigate + About
vibium go https://var.parts/ && vibium map
vibium click @e3 && vibium wait load && vibium url
# Expected: https://var.parts/about

# Product detail
vibium go https://var.parts/product/12 && vibium wait load && vibium url
# Expected: https://var.parts/product/12

# Add to cart
vibium go https://var.parts/
vibium map
vibium click @e8 && vibium wait text "Added to cart"
vibium click @e11 && vibium wait text "Added to cart"

# Cart
vibium map
vibium click @e7  # cart badge showing "2"
vibium wait load && vibium text
# Expected: "2 items in your cart", total = ⚡239.98

# Checkout
vibium map
vibium click @e11 && vibium wait load
vibium map
vibium fill @e7 "VAR-999" && vibium fill @e8 "Bay 99-Z"
vibium click @e14 && vibium wait load

# Pay
vibium map
vibium click @e10 && vibium wait text "Payment Received!"
# Expected: confirmation page

# Return to shop
vibium map
vibium click @e7 && vibium wait load && vibium url
# Expected: https://var.parts/
```

## Args

Pass a specific flow name to run just that flow:
- `navigation` — test nav links only
- `product` — test product listing and detail
- `cart` — test cart management
- `checkout` — test full checkout + payment
- `smoke` — abbreviated happy-path run

Default (no args): run all flows.
