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
| [demo.prestashop.com](https://demo.prestashop.com/) | PrestaShop | Server-side (per subdomain) | Bank wire / COD / Check | Store in iframe; subdomains expire in ~2 min; `vibium go` to subdomain pages deadlocks daemon (B3) — use `eval location.href`; **add-to-cart button permanently disabled** after PS JS init — cart untestable |
| [coffee-cart.app](https://coffee-cart.app/) | Custom Vue SPA | In-memory | None (email link) | Products are divs; right-click dialog; promo popup on 3rd item |
| [automationteststore.com](https://automationteststore.com/) | AbanteCart | Server-side | Cash on Delivery | Add to Cart is `<a>` not button; qty input has hash-based name |
| [academybugs.com](https://academybugs.com/) | Custom PHP | Partial (buggy) | None | 25 planted bugs; cart layout broken by design; dismiss modal + cookie banner first |
| [bookcart.azurewebsites.net](https://bookcart.azurewebsites.net/) | Angular + Azure | Server-side | None (demo) | Backend hibernates — reload once and wait 30s; silent login failure |
| [magento.softwaretestingboard.com](https://magento.softwaretestingboard.com/) | Magento | — | — | **DOWN** — Cloudflare 526 SSL error as of 2026-04-22 |
| [shop.polymer-project.org](https://shop.polymer-project.org/) | Polymer Web Components | Server-side | None (demo) | All UI in shadow DOM — `vibium map` returns nothing; use eval+coords for all interaction |
| [practicesoftwaretesting.com](https://practicesoftwaretesting.com/) | Angular | Server-side | None (demo) | Add to cart works without login; Login is `input[type=submit]`; shared account may be locked; Angular async delay after filters — sleep 1500 |
| [qa-practice.razvanvancea.ro](https://qa-practice.razvanvancea.ro/) | Custom HTML/JS | In-memory (lost on reload) | None (demo) | Login: `admin@admin.com` / `admin123`; ADD TO CART is CSS uppercase — use map refs; pre-stub `alert`/`confirm` via eval before clicking buttons that trigger dialogs; checkout DOM-toggle (form parent display:none → block) |

---

## Cross-site gotchas

### Hibernating backends (Azure / Heroku)
Demo sites on Azure App Service or Heroku free tier hibernate after inactivity. If the product list loads empty with no error:
```bash
vibium sleep 5000 && vibium reload && vibium wait load --timeout 20000
```
Wait up to 30 seconds. If products still don't appear after one reload, skip the site and log as infrastructure issue.

### Web Components / shadow DOM sites
`vibium map`, `vibium click`, `vibium fill`, and `vibium find` are all blocked by shadow DOM boundaries. Sites like **Polymer Shop** render their entire UI inside custom elements. Use `eval + shadowRoot` to traverse and `getBoundingClientRect()` + `vibium mouse click x y` to interact:
```bash
# Find element and get coordinates
vibium eval 'JSON.stringify(document.querySelector("shop-app").shadowRoot.querySelector("shop-button")?.getBoundingClientRect())'
# → {"x":720,"y":734,"width":186,"height":36}
vibium mouse click 813 752
# Read state
vibium eval 'document.querySelector("shop-app").shadowRoot.querySelector("shop-cart")?.shadowRoot?.querySelector(".subtotal")?.textContent?.trim()'
```
Navigate between pages via `vibium go <direct-url>` — shadow DOM links are not clickable through vibium.

### CSS text-transform — button text mismatch
Some sites render buttons as uppercase via CSS while DOM text is mixed-case. `vibium find text "ADD TO CART"` silently fails — use `vibium map` refs instead. Affects: **AcademyBugs**, **QA Practice**.

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

### PrestaShop demo — ephemeral subdomains and iframe wrapper
`demo.prestashop.com` wraps the actual store in a `#framelive` iframe. Navigate to the homepage with `vibium go`, then use JS navigation for all subsequent page transitions — `vibium go` to subdomain pages deadlocks the daemon (B3 pattern):
```bash
vibium go https://demo.prestashop.com/ && vibium wait load && vibium sleep 5000
vibium eval 'document.querySelector("#framelive")?.src'
# → https://gigantic-appliance.demo.prestashop.com/en/...
vibium go "https://gigantic-appliance.demo.prestashop.com/en/" && vibium wait load  # homepage only

# All further navigation — use eval, not vibium go:
vibium eval 'location.href = "https://gigantic-appliance.demo.prestashop.com/1-1-hummingbird-printed-t-shirt.html"'
vibium wait load --timeout 10000
```
The inner subdomain expires in ~2 min of active testing. If you hit "Oops... couldn't find the shop", fetch a new subdomain and restart. Subdomains cannot be reused after expiry.

### PrestaShop qty update — Enter required
Filling the qty input alone does nothing. Always follow with Enter:
```bash
vibium fill "input[name='product-quantity-spin']" "2"
vibium press Enter "input[name='product-quantity-spin']"
vibium sleep 2000
```

### PrestaShop remove — target the anchor, not the div
`vibium find text "Remove"` returns an outer wrapper `<div>` that fails with "element is obscured". Target the inner link directly:
```bash
vibium find "a.js-remove-from-cart" && vibium click @e1 && vibium sleep 2000
# Reload to confirm — the DOM update is async
vibium reload && vibium wait load
```

### PrestaShop address fields — scroll before fill
Address form fields have zero computed size until scrolled into view:
```bash
vibium scroll into-view "#field-address1" && vibium sleep 300
vibium fill "#field-address1" "123 Test Street"
vibium fill "#field-postcode" "75001"
vibium fill "#field-city" "Paris"
```
Submit buttons suffer the same issue — use eval.click():
```bash
vibium eval 'document.querySelector("form button[type=submit]").click()'
```

### PrestaShop country dropdown — numeric IDs
Like OpenCart, PrestaShop uses numeric option values. Inspect before selecting:
```bash
vibium eval 'JSON.stringify([...document.querySelector("#field-id_country").options].map(o => ({v:o.value,t:o.text})))'
# → [{"v":"","t":"-- please choose --"},{"v":"8","t":"France"},{"v":"21","t":"United States"}]
vibium select "#field-id_country" "8"
```

### SPA in-memory carts — use UI nav, not vibium go
Some SPAs (Vue, React) keep cart state in component memory. A `vibium go <cart-url>` triggers a full page reload and wipes the cart. Always navigate via the UI link:
```bash
vibium find "a[href='/cart']" && vibium click @e1 && vibium wait load
# NOT: vibium go https://coffee-cart.app/cart
```

### Product cards that aren't buttons or links
Some apps use `<div>` elements as clickable product cards with `data-test` or `aria-label` attributes instead of `<button>` or `<a>`. `vibium map` won't surface them — target by attribute:
```bash
vibium find "[data-test='Espresso']" && vibium click @e1
# Underscores for spaces: Flat_White, Cafe_Latte, Espresso_Con_Panna
```
To right-click (contextmenu) a product card:
```bash
vibium eval 'document.querySelector("[data-test=\"Espresso\"]").dispatchEvent(new MouseEvent("contextmenu", {bubbles:true}))'
vibium sleep 500 && vibium map
# Then click the Yes/No dialog buttons
```

### AbanteCart — Add to Cart is an anchor element
AbanteCart renders Add to Cart as `<a class="cart">`, not `<button>`. Clicking navigates directly to the cart page (no toast or modal). Some products use "Call To Order" and have no `a.cart` — skip them.
```bash
vibium find "a.cart" && vibium click @e1
vibium wait load && vibium url  # → /index.php?rt=checkout/cart
```

### AbanteCart — qty input name contains a hash
The quantity input name is `quantity[productid:md5hash]`. Get the exact name before filling:
```bash
vibium eval 'JSON.stringify([...document.querySelectorAll("input")].filter(i => i.name.startsWith("quantity")).map(i => i.name))'
# → ["quantity[53:b1a0e11451071a263d5a530074cc3395]"]
vibium fill "input[name='quantity[53:b1a0e11451071a263d5a530074cc3395]']" "3"
vibium find "button[title='Update']" && vibium click @e1 && vibium wait load
```

### AbanteCart — remove via URL href selector
Remove link is an `<a>` with `remove=productid:hash` in its href. Target it by partial href match:
```bash
vibium find "a[href*='remove=']" && vibium click @e1 && vibium wait load
```

### Cloudflare-protected demo sites
Some popular demo hosts put Cloudflare in front. Two challenge types encountered:
- **`demo.nopcommerce.com`** — Turnstile "managed" (checkbox widget, no iframe in DOM). Cannot be bypassed by automation; browser fingerprinting blocks headless Chrome unconditionally.
- **`demo.opencart.com`** — JS challenge (spinner). Also fails — browser entropy check never passes in headless mode.

If a site shows "Performing security verification" with a spinning loader or "Verify you are human" checkbox and never proceeds, it's Cloudflare-protected. Skip it and find an unprotected instance.

### Cart state persistence
- **var.parts**: In-memory — cart is lost on any direct URL navigation. Always use the header cart badge.
- **coffee-cart.app**: In-memory — use `vibium find "a[href='/cart']" && vibium click @e1`, never `vibium go`.
- **qa-practice.razvanvancea.ro**: In-memory — entire session (cart + login) is lost on page reload. Never reload mid-flow.
- **Shopify**: Server-side session — direct `/cart` URL is safe regardless of navigation path.
- **saucedemo.com**: Server-side session — direct `/cart.html` URL is safe.
- **OpenCart (lambdatest)**: Server-side session — direct cart URL is safe.
- **AbanteCart (automationteststore)**: Server-side session — direct `/index.php?rt=checkout/cart` is safe.
- **PrestaShop (demo)**: Server-side session per ephemeral subdomain — direct `/cart?action=show` works within the same instance, but the subdomain expires in ~2 min of active testing.

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

## Site: coffee-cart.app

**URL:** https://coffee-cart.app/  
**Platform:** Custom Vue.js SPA  
**Products:** 9 coffees, $7–$19 — Espresso, Espresso Macchiato, Cappuccino, Mocha, Flat White, Americano, Cafe Latte, Espresso Con Panna, Cafe Breve  
**Cart:** In-memory — wiped on full page reload; always navigate via `<a href='/cart'>` link  
**Checkout:** Modal overlay, not a separate page; no real payment gateway

**Special behaviors:**
- Promo popup fires on every 3rd item added: discounted upsell with Yes/No
- Double-click `<h4>` name translates to Chinese; double-click again reverts
- Right-click on any cup opens an "Add X to the cart?" confirm dialog
- `?breakable=1` param: random intermittent add-to-cart errors (for DevTools practice)
- `?ad=1` param: injects ad banners that slow page load

**No known bugs** — all core flows pass.

### Test flows

#### 1. Navigation
```bash
vibium go https://coffee-cart.app/ && vibium map
# @e1 Menu (/), @e2 Cart (/cart), @e3 GitHub (/github — internal info page, not external)
vibium click @e2 && vibium wait load && vibium url
# Expected: https://coffee-cart.app/cart
vibium click @e1 && vibium wait load && vibium url
# Expected: https://coffee-cart.app/
vibium click @e3 && vibium wait load && vibium url
# Expected: https://coffee-cart.app/github
vibium text
# Verify: page describes special behaviors (right-click, double-click, promo, breakable, etc.)
```

#### 2. Product listing
```bash
vibium go https://coffee-cart.app/ && vibium wait load
vibium text
# Verify: 9 product names and prices visible
vibium eval 'document.querySelectorAll("div.cup-body").length'
# Expected: 9
```
Note: products are `div.cup-body[data-test]` elements — `vibium map` returns only nav links and the checkout button.

#### 3. Add to cart (click)
```bash
vibium find "[data-test='Espresso']" && vibium click @e1 && vibium sleep 800
vibium eval '[...document.querySelectorAll("li")].find(el => el.textContent.includes("cart"))?.textContent.trim()'
# Expected: "cart (1)"
vibium eval 'document.querySelector("button[data-test=checkout]")?.textContent.trim()'
# Expected: "Total: $10.00"
```

#### 4. Add to cart (right-click dialog)
```bash
vibium eval 'document.querySelector("[data-test=\"Espresso_Macchiato\"]").dispatchEvent(new MouseEvent("contextmenu", {bubbles:true}))'
vibium sleep 500 && vibium screenshot -o /tmp/coffee_rclick.png
# Verify: "Add Espresso Macchiato to the cart?" dialog visible with Yes / No buttons

# Accept:
vibium find role button --name "Yes" && vibium click @e1 && vibium sleep 500
# Expected: cart count +1

# Or dismiss:
vibium find role button --name "No" && vibium click @e1 && vibium sleep 500
# Expected: cart count unchanged
```

#### 5. Cart management
```bash
# Navigate via SPA link — never vibium go /cart
vibium find "a[href='/cart']" && vibium click @e1 && vibium wait load
vibium text
# Verify: items listed with "Item / Unit / Total" table, grand total shown

vibium map
# @eN [button] "Add one Espresso", "Remove one Espresso", "Remove all Espresso"

# Increase qty
vibium find role button --name "Add one Espresso" && vibium click @e1 && vibium sleep 500
vibium text
# Verify: "x 2", subtotal doubled, total updated

# Decrease qty
vibium find role button --name "Remove one Espresso" && vibium click @e1 && vibium sleep 500
# Verify: back to x 1

# Remove all
vibium find role button --name "Remove all Espresso" && vibium click @e1 && vibium sleep 500
# Verify: Espresso row gone, total updated
```

#### 6. Empty cart
```bash
# After clearing all items:
vibium text
# Verify: "No coffee, go add some."
vibium map
# Verify: "Proceed to checkout" button absent (no @eN for it)
```

#### 7. Promo popup (every 3rd item)
```bash
vibium find "[data-test='Mocha']" && vibium click @e1 && vibium sleep 400
vibium find "[data-test='Americano']" && vibium click @e1 && vibium sleep 400
vibium find "[data-test='Flat_White']" && vibium click @e1 && vibium sleep 1500
vibium screenshot -o /tmp/coffee_promo.png
# Verify: "It's your lucky day! Get an extra cup of [X] for $4."

# Accept promo:
vibium find role button --name "Yes, of course!" && vibium click @e1 && vibium sleep 500
# Expected: cart count +1 (discounted item added)

# Skip promo (reset and repeat for skip path):
vibium find role button --name "Nah, I'll skip." && vibium click @e1 && vibium sleep 500
# Expected: cart count unchanged
```

#### 8. Checkout form validation
```bash
vibium find role button --name "Proceed to checkout" && vibium click @e1 && vibium sleep 800
# Verify: "Payment details" modal appears
vibium click "#submit-payment" && vibium sleep 500
# Verify: "Please fill out this field." on Name (HTML5 native tooltip)

vibium fill "#name" "Test User" && vibium click "#submit-payment" && vibium sleep 500
# Verify: "Please fill out this field." on Email

vibium fill "#email" "notanemail" && vibium click "#submit-payment" && vibium sleep 500
# Verify: "Please include an '@' in the email address."
```

#### 9. Full checkout (happy path)
```bash
vibium go https://coffee-cart.app/ && vibium wait load
vibium find "[data-test='Espresso']" && vibium click @e1 && vibium sleep 500
vibium find "[data-test='Mocha']" && vibium click @e1 && vibium sleep 500

vibium find role button --name "Proceed to checkout" && vibium click @e1 && vibium sleep 800

vibium fill "#name" "Test User"
vibium fill "#email" "test@example.com"
vibium click "#submit-payment" && vibium sleep 2000

vibium screenshot -o /tmp/coffee_done.png
vibium text
# Verify: "Thanks for your purchase. Please check your email for payment."
# Verify: cart badge shows "cart (0)"
# Verify: a random product is highlighted in gold (post-order promo recommendation)
```

#### 10. Double-click translation
```bash
vibium go https://coffee-cart.app/ && vibium wait load
vibium eval 'document.querySelector("h4").textContent.trim()'
# Expected: "Espresso $10.00"

vibium dblclick "h4" && vibium sleep 500
vibium eval 'document.querySelector("h4").textContent.trim()'
# Expected: "特浓咖啡 $10.00"

vibium dblclick "h4" && vibium sleep 500
vibium eval 'document.querySelector("h4").textContent.trim()'
# Expected: "Espresso $10.00" (reverted)
```

#### Quick smoke test
```bash
vibium go https://coffee-cart.app/ && vibium wait load

# Add two items
vibium find "[data-test='Espresso']" && vibium click @e1 && vibium sleep 400
vibium find "[data-test='Americano']" && vibium click @e1 && vibium sleep 400
vibium eval 'document.querySelector("button[data-test=checkout]")?.textContent.trim()'
# Expected: "Total: $17.00"

# Navigate to cart via SPA link
vibium find "a[href='/cart']" && vibium click @e1 && vibium wait load
vibium text
# Verify: Espresso $10.00, Americano $7.00, Total: $17.00

# Checkout
vibium find role button --name "Proceed to checkout" && vibium click @e1 && vibium sleep 800
vibium fill "#name" "Test User" && vibium fill "#email" "test@example.com"
vibium click "#submit-payment" && vibium sleep 2000
vibium text
# Verify: "Thanks for your purchase..."
```

---

## Site: demo.prestashop.com

**URL:** https://demo.prestashop.com/ (inner store: `*.demo.prestashop.com/en/`)  
**Platform:** PrestaShop  
**Products:** Hummingbird printed t-shirt (€22.94 sale / €28.68), Brown bear sweater (€34.46), framed posters. Categories: Clothes, Accessories, Art.  
**Product URLs:** `/{subdomain}/1-1-hummingbird-printed-t-shirt.html`, `/2-9-brown-bear-printed-sweater.html`  
**Cart:** Server-side session per ephemeral subdomain — expires in ~2 min of active testing  
**Checkout:** 4-step: Personal Information → Addresses → Shipping method → Payment

**Known bugs:**
- **Add to cart button permanently disabled** — `[data-button-action=add-to-cart]` becomes `disabled=true` after PrestaShop JS initializes. Affects product detail pages AND homepage product cards. Click via map ref succeeds (no error) but AJAX fails silently and cart stays at 0. Direct POST to `/cart` returns HTML, not JSON. Cart and checkout flows are untestable.
- Cart qty Increase button (`+`) in cart view is also permanently `disabled`
- Empty cart checkout (`/en/order`) redirects to `/cart?action=show` with "There are no more items in your cart" — expected behavior, but checkout form validation is untestable due to add-to-cart bug above
- "Your address is incomplete" error can appear even with a valid complete address (demo instance isolation issue)

**Automation quirks:**
- Store is inside a `#framelive` iframe — navigate to homepage with `vibium go`, then use `eval 'location.href = "..."'` + `vibium wait load` for all subsequent navigation (`vibium go` to subdomain pages deadlocks daemon — B3)
- Subdomains expire in ~2 min; get a fresh one from the iframe src if the page shows "couldn't find the shop"
- Product detail variant selectors: Size = `#input_1_1` (`name="group[1]"`, values `1`–`4`); Color = `input[name="group[2]"]` (val=`8` White, val=`11` Black)
- Address form fields are off-screen and report zero-size — scroll into view before filling
- Submit buttons also zero-size — use `eval 'form button[type=submit].click()'`
- Remove link is inside a wrapper div; target `a.js-remove-from-cart` directly
- Qty update requires fill + Enter (fill alone does nothing)
- Country dropdown uses numeric IDs (`8` = France, `21` = United States)

### Test flows

#### 1. Get a live instance
```bash
vibium go https://demo.prestashop.com/ && vibium wait load && vibium sleep 5000
vibium eval 'document.querySelector("#framelive")?.src'
# Copy the inner URL, e.g. https://gigantic-appliance.demo.prestashop.com/en/
vibium go "https://gigantic-appliance.demo.prestashop.com/en/" && vibium wait load
# IMPORTANT: all subsequent navigation must use eval location.href (vibium go deadlocks — B3)
```

#### 2. Navigation
```bash
# Use eval location.href — vibium click on nav links and vibium go both deadlock (B3)
STORE="https://gigantic-appliance.demo.prestashop.com"
vibium eval "location.href = '${STORE}/en/3-clothes'" && vibium wait load && vibium url
# Expected: /3-clothes
vibium eval "location.href = '${STORE}/en/6-accessories'" && vibium wait load && vibium url
# Expected: /6-accessories
vibium eval "location.href = '${STORE}/en/9-art'" && vibium wait load && vibium url
# Expected: /9-art
```

#### 3. Product listing
```bash
STORE="https://gigantic-appliance.demo.prestashop.com"
vibium eval "location.href = '${STORE}/en/3-clothes'" && vibium wait load
vibium text
# Verify: "Hummingbird printed t-shirt", "Hummingbird printed sweater", prices visible
vibium map --selector ".products"
# Verify: "Add to cart" buttons present for each product (NOTE: buttons may show as disabled — see bug)
```

#### 4. Product detail
```bash
STORE="https://gigantic-appliance.demo.prestashop.com"
vibium eval "location.href = '${STORE}/1-1-hummingbird-printed-t-shirt.html'" && vibium wait load
vibium text
# Verify: name, sale price (€22.94), regular price (€28.68), -20% badge
# Verify: Size selector (S/M/L/XL via #input_1_1), Color radios (White val=8 / Black val=11)
```

#### 5. Add to cart
```bash
# ⚠ BUG: Add to cart button is permanently disabled after PS JS init.
# [data-button-action=add-to-cart] becomes disabled=true on page load.
# Click via ref succeeds but AJAX fails silently — cart stays at 0.
# This flow is currently untestable on the live demo instance.
vibium eval "document.querySelector('[data-button-action=add-to-cart]').disabled"
# → true (confirms bug)
```

#### 6. Cart management
```bash
# Qty update — fill + Enter (fill alone is ignored)
vibium fill "input[name='product-quantity-spin']" "2"
vibium press Enter "input[name='product-quantity-spin']" && vibium sleep 2000
vibium eval 'document.querySelector("[class*=total]")?.textContent.trim().replace(/\s+/g," ")'
# Verify: "2 items €45.89 ... Total (tax incl.) €45.89"

# Remove — target the anchor, not the div
vibium find "a.js-remove-from-cart" && vibium click @e1 && vibium sleep 2000
vibium reload && vibium wait load
# Verify: redirected to home, cart badge = 0
```

#### 7. Empty cart
```bash
STORE="https://gigantic-appliance.demo.prestashop.com"
vibium eval "location.href = '${STORE}/cart?action=show'" && vibium wait load
vibium text
# Expected: "There are no more items in your cart" (cart page shown correctly)
vibium eval "location.href = '${STORE}/en/order'" && vibium wait load && vibium url
# Expected: redirects to /cart?action=show (checkout correctly blocked with empty cart)
```

#### 8. Checkout form validation
```bash
# ⚠ BLOCKED: Add to cart is disabled — cart cannot be populated, checkout unreachable.
# If add-to-cart is fixed in a future instance, use:
# vibium eval 'document.querySelector("button[type=submit]").click()' && vibium sleep 1500
# vibium eval 'JSON.stringify([...document.querySelectorAll(":invalid")].map(el => ({name: el.name, msg: el.validationMessage})))'
# Expected: firstname, lastname, email, psgdpr, customer_privacy all invalid
```

#### 9. Full checkout (happy path)
```bash
# Step 1 — Personal Information
vibium fill "#field-firstname" "Test"
vibium fill "#field-lastname" "User"
vibium fill "#field-email" "testuser@example.com"
vibium eval 'document.querySelector("#field-psgdpr").click()'
vibium eval 'document.querySelector("#field-customer_privacy").click()'
vibium eval 'document.querySelector("button[type=submit]").click()' && vibium sleep 3000
# Expected: proceeds to Addresses step

# Step 2 — Addresses
vibium scroll into-view "#field-address1" && vibium sleep 300
vibium fill "#field-address1" "123 Test Street"
vibium fill "#field-postcode" "75001"
vibium fill "#field-city" "Paris"
vibium eval 'JSON.stringify([...document.querySelector("#field-id_country").options].map(o=>({v:o.value,t:o.text})))'
# → use numeric value for France
vibium select "#field-id_country" "8" && vibium sleep 500
vibium eval 'document.querySelector("form button[type=submit]").click()' && vibium sleep 3000
# Expected: proceeds to Shipping method step

# Step 3 — Shipping method
# "Click and collect" (Free) is pre-selected
vibium find text "Continue to Payment" && vibium click @e1 && vibium sleep 2000
# Expected: proceeds to Payment step

# Step 4 — Payment
vibium eval 'document.querySelector("#payment-option-2").click()'  # Cash on Delivery
vibium eval 'document.querySelector("[name=\"conditions_to_approve[terms-and-conditions]\"]").click()'
vibium scroll into-view "#payment-confirmation" && vibium sleep 300
vibium find role button --name "Place Order" && vibium click @e1 && vibium sleep 5000
vibium url
# Expected: /order-confirmation or /order-detail
vibium text
# Verify: "Your order is confirmed", order number shown, cart cleared
```

#### Quick smoke test
```bash
# Get fresh instance (use eval for all navigation after homepage — vibium go deadlocks B3)
vibium go https://demo.prestashop.com/ && vibium wait load && vibium sleep 5000
STORE=$(vibium eval 'document.querySelector("#framelive")?.src' | grep -o 'https://[^/]*')
vibium go "$STORE/en/" && vibium wait load

# Verify structure
vibium map
# Expected: 80 interactive elements; Clothes/Accessories/Art nav; 4 featured products

# Check add-to-cart button state (known bug)
vibium eval 'document.querySelector("[data-button-action=add-to-cart]")?.disabled'
# → true = bug still present; false = may be fixed, proceed to test flow 5

# Navigate to product detail (use eval, not click)
vibium eval "location.href = '${STORE}/1-1-hummingbird-printed-t-shirt.html'" && vibium wait load
vibium eval 'JSON.stringify({name: document.querySelector("h1")?.textContent?.trim(), price: document.querySelector(".product__price")?.textContent?.trim()})'
# Verify: name "Hummingbird printed t-shirt", price "€22.94"
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

---

## Site: automationteststore.com

**URL:** https://automationteststore.com/
**Platform:** AbanteCart (OpenCart fork)
**Currency:** USD
**Cart persistence:** Server-side session

### Key behaviors

- Add to Cart is `<a class="cart">`, not a button — clicking navigates directly to cart page
- Some products (e.g. Infinitive Mascara, product_id=54) are "Call To Order" with no Add to Cart
- Reliable test product: **Tropiques Minerale Loose Bronzer** (Makeup → product_id=53, $38.50)
- Cart qty input name: `quantity[productid:md5hash]` — inspect before filling
- Remove: `a[href*='remove=']`
- Guest checkout: radio button `input[name=account][value=guest]`
- Country/state: same numeric IDs as OpenCart (223 = US, 3669 = Texas)
- No partial reload after country select — no Terms reset needed
- Tax (8.5%) appears at confirmation stage
- Success page correctly redirects to homepage on direct navigation

### Smoke test

```bash
vibium go https://automationteststore.com/index.php?rt=product/product&product_id=53
vibium find "a.cart" && vibium click @e1 && vibium wait load
vibium url  # → /index.php?rt=checkout/cart
vibium text  # Verify: Tropiques Minerale Loose Bronzer, $38.50

# Guest checkout
vibium eval 'document.querySelector("a[href*=\"checkout/shipping\"]").click()' && vibium wait load
vibium url  # → /index.php?rt=account/login
vibium eval 'document.querySelector("input[name=account][value=guest]").click()'
vibium click "button[title='Continue']" && vibium wait load
vibium url  # → /index.php?rt=checkout/guest_step_1
```

### Flow 1 — Navigation

```bash
vibium go https://automationteststore.com/ && vibium map
# Nav links: Apparel & Accessories, Makeup, Skincare, Fragrance, Men, Hair Care, Books
vibium click "[href*='category_id=36']" && vibium wait load  # Apparel & Accessories
vibium url  # → ?rt=product/category&path=36
vibium back && vibium wait load
vibium click "[href*='category_id=44']" && vibium wait load  # Makeup
vibium url  # → ?rt=product/category&path=44
```

### Flow 2 — Product listing

```bash
vibium go "https://automationteststore.com/index.php?rt=product/category&path=44" && vibium wait load
vibium text
# Verify: product names + prices visible
vibium eval 'document.querySelectorAll("a[title=\"Add to Cart\"]").length'
# Verify: > 0 (note: some products show "OUT OF STOCK" or "Call To Order")
```

### Flow 3 — Product detail

```bash
vibium go "https://automationteststore.com/index.php?rt=product/product&product_id=53"
vibium wait load && vibium text
# Verify: "Tropiques Minerale Loose Bronzer", "$38.50", Color dropdown, Add to Cart link
vibium find "a.cart"
# Verify: found (not "Call To Order")
```

### Flow 4 — Add to cart

```bash
vibium go "https://automationteststore.com/index.php?rt=product/product&product_id=53"
vibium wait load
vibium find "a.cart" && vibium click @e1 && vibium wait load
vibium url  # → /index.php?rt=checkout/cart
vibium text  # Verify: product name, $38.50 in cart
```

### Flow 5 — Cart management

```bash
# Get exact qty input name
vibium eval 'JSON.stringify([...document.querySelectorAll("input")].filter(i=>i.name.startsWith("quantity")).map(i=>({name:i.name,value:i.value})))'
# → [{"name":"quantity[53:b1a0e11451071a263d5a530074cc3395]","value":"1"}]

vibium fill "input[name='quantity[53:b1a0e11451071a263d5a530074cc3395]']" "3"
vibium find "button[title='Update']" && vibium click @e1 && vibium wait load
vibium text  # Verify: "3 ITEMS - $115.50", Total: $117.50

# Remove
vibium find "a[href*='remove=']" && vibium click @e1 && vibium wait load
vibium text  # Verify: "Your shopping cart is empty!"
```

### Flow 6 — Empty cart

```bash
# Cart page shows empty state
vibium url && vibium text  # "Your shopping cart is empty!"

# Checkout from empty cart — redirect to login → then to cart (blocked)
vibium go "https://automationteststore.com/index.php?rt=checkout/shipping" && vibium wait load
vibium url  # → /index.php?rt=checkout/cart (redirected, not let through)
```

### Flow 7 — Checkout form validation

```bash
# Add item first
vibium go "https://automationteststore.com/index.php?rt=product/product&product_id=53"
vibium find "a.cart" && vibium click @e1 && vibium wait load

# Navigate to guest checkout step 1
vibium eval 'document.querySelector("a[href*=\"checkout/shipping\"]").click()' && vibium wait load
vibium eval 'document.querySelector("input[name=account][value=guest]").click()'
vibium click "button[title='Continue']" && vibium wait load

# Submit without filling any fields
vibium click "button[title='Continue']" && vibium wait load
vibium eval 'JSON.stringify([...document.querySelectorAll(".help-block")].map(el=>el.textContent.trim()).filter(t=>t))'
# Verify: errors for First Name, Last Name, E-Mail, Address 1, City, Region/State, ZIP
```

### Flow 8 — Full checkout (happy path)

```bash
# Fill billing/shipping form
vibium fill "#guestFrm_firstname" "Test"
vibium fill "#guestFrm_lastname" "User"
vibium fill "#guestFrm_email" "guest_test@example.com"
vibium fill "#guestFrm_telephone" "12025550100"
vibium fill "#guestFrm_address_1" "123 Test Street"
vibium fill "#guestFrm_city" "Austin"
vibium eval 'JSON.stringify([...document.querySelector("#guestFrm_country_id").options].filter(o=>o.text.includes("United States")).map(o=>o.value))'
# → ["223"]
vibium select "#guestFrm_country_id" "223" && vibium wait load && vibium sleep 1000
vibium select "#guestFrm_zone_id" "3669"
vibium fill "#guestFrm_postcode" "78701"
vibium click "button[title='Continue']" && vibium wait load
vibium url  # → /index.php?rt=checkout/guest_step_3

# Confirmation page — review and confirm
vibium text ".contentpanel"
# Verify: Tropiques Minerale, $38.50, Flat Shipping Rate $2.00, Tax $3.27, Total $43.77
vibium click "#checkout_btn" && vibium wait load
vibium url  # → /index.php?rt=checkout/success
vibium text "h1"  # → "YOUR ORDER HAS BEEN PROCESSED!"

# Verify cart cleared
vibium eval 'document.querySelector(".items-count")?.textContent'  # → "0 ITEMS"

# Verify success page secured
vibium go "https://automationteststore.com/index.php?rt=checkout/success" && vibium wait load
vibium url  # → homepage (redirected, not order context shown)
```
