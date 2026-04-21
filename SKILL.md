---
name: cart-patrol
description: Run e2e tests on the var.parts robot parts storefront. Tests navigation, product listing, product detail, cart management (add/remove/clear/quantity), checkout form validation, and payment flows (Pay Here + Other Device). Use when asked to test var.parts or run the cart-patrol suite.
---

# var-parts-test

Test the e2e user flow on https://var.parts/ using vibium browser automation.

Refer to README.md in this directory for the full test suite with vibium commands.

## How to run

When invoked, check the ARGUMENTS value:
- No args or `all`: run all flows in sequence
- `smoke`: run the quick smoke test only
- Named flow (`navigation`, `product`, `cart`, `checkout`, `payment`): run that flow only

For each flow, report pass/fail with what you observed. Note unexpected behavior as a bug. End with a summary: X/Y flows passed, any failures.

## Key site facts

- URL: https://var.parts/
- 12 products on homepage in categories: SWAG, ACCESSORIES, HEAD UNITS, CHASSIS, WIRING, HARDWARE
- Cart is in-memory — always navigate to cart via UI (header cart badge), never `vibium go /cart` after adding items or the cart will appear empty
- Required checkout fields: Unit Designation, Service Bay
- Optional checkout fields: Sector, Station
- Validation error text: "Missing address" + "Unit designation and service bay are required."
- Payment tabs: "Pay Here" (Pay Now button) and "Other Device" (QR + copy link)
- Copy link button feedback: "Link copied to clipboard" (not "Copied")
- After Pay Now: shows "Processing..." then navigates to "Payment Received!" confirmation
- Post-payment cart is cleared; "Back to Shop" returns to /

## Known behavior (not bugs)

- Back to Details from payment page retains form values internally even if inputs appear blank
- Direct navigation to /checkout with empty cart shows "Nothing to check out"
