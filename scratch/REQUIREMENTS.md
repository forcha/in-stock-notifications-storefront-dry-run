# Requirements: Out-of-Stock Notification Form

## Overview

Add a subscription form to the Product Detail Page (PDP) that allows shoppers to
register for back-in-stock email notifications. The form is displayed only when
the product is out of stock and appears beneath the Add to Cart and Wishlist buttons.

## API Contract

- **Base URL:** `https://1899289-dryrunpd26bcn11-stage.adobeioruntime.net/api/v1/web/stock-notifications`
- **Check status:** `GET /check-status/{sku}?email={email}` → `{ sku, email, subscribed }`
- **Subscribe/Unsubscribe:** `POST /subscribe/{sku}` with `{ email, subscribed: true|false }`
- No authentication required from the frontend (proxy handles credentials)
- CORS: `Access-Control-Allow-Origin: *`

## Placement

- PDP, below the Add to Cart and Wishlist buttons
- Visible only when product is out of stock

## Confirmed Requirements

| # | Question | Answer |
|---|----------|--------|
| Q1 | Email pre-fill for logged-in users? | Yes — pre-fill from account email |
| Q2 | Variant-specific or parent SKU? | Variant-specific SKU; re-check on variant switch |
| Q3 | Available to guests and logged-in users? | All users |
| Q4 | Persist subscription state? | Yes — remember via localStorage |
| Q5 | Unsubscribe flow? | Yes — support "Cancel notification" |
| Q6 | Styling & button label? | Match design tokens; label = "Notify Me" |
| Q7 | Commerce tier? | Adobe Commerce SaaS (ACCS) |

## Technical Context (Discovered)

- PDP block: `blocks/product-details/product-details.js` / `.css`
- Drop-in events used:
  - `pdp/data` → `ProductModel` with `sku` (parent), `variantSku` (resolved variant), `inStock: boolean|null`
  - `pdp/values` → fires on option change; call `pdpApi.getProductConfigurationValues()` for current `sku`
- Auth API: `getCustomerData()` from `@dropins/storefront-auth/api.js`
  returns `{ data: { customer: { email, firstname, lastname } } }`
- Design tokens defined in `styles/styles.css` (color, spacing, border-radius, shadow)

---

## Phase 1: Complete ✅
Date: 2026-03-20

## Phase 2: Architectural Plan Presented
Date: 2026-03-20

## Phase 2: Complete ✅
User Approved: Yes
Approval Date: 2026-03-20

## Phase 3: Implementation Approach Selected
Approach: Option B (Direct Implementation)
Selection Date: 2026-03-20

## Phase 4: Implementation Started
Date: 2026-03-20

## Phase 4: Implementation Complete
Date: 2026-03-20
Files Modified:
- blocks/product-details/product-details.js
- blocks/product-details/product-details.css
