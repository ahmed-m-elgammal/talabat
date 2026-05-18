# 13 — UI/UX Specifications

## 1. Overview

Talabat's UI/UX design follows the **MM3 design system**, a comprehensive design language developed by Delivery Hero for its food delivery platforms worldwide. The system emphasizes clarity, speed, and cultural sensitivity across the MENA region, supporting both LTR (English) and RTL (Arabic) layouts with 9 country-specific currency and language configurations. The design philosophy prioritizes **task completion speed** — users should be able to place an order with minimal taps, aided by intelligent defaults, saved preferences, and progressive disclosure of complex options.

This specification covers the visual design system, interaction patterns, accessibility requirements, and per-screen UX specifications for the Talabat mobile application.

---

## 2. Design System Foundation

### 2.1 Brand Colors

| Color | Hex | Usage |
|-------|-----|-------|
| Talabat Orange (Primary) | `#FF5A00` | Primary CTA buttons, active states, brand accent |
| Talabat Dark | `#1A1A2E` | Primary text, navigation bars |
| Talabat Gray | `#6B7280` | Secondary text, inactive states |
| Talabat Light Gray | `#F3F4F6` | Background, dividers |
| Success Green | `#10B981` | Delivered status, positive feedback |
| Warning Amber | `#F59E0B` | Busy status, caution messages |
| Error Red | `#EF4444` | Errors, cancellation, out of stock |
| Pro Gold | Custom | Pro membership badges and labels |
| TGO Blue | Custom | Guaranteed Order badges |

### 2.2 Typography Scale

**English (TTCommonsPro):**

| Style | Weight | Size | Line Height | Usage |
|-------|--------|------|-------------|-------|
| Display | 900 | 34sp | 40sp | Onboarding headlines |
| H1 | 800 | 28sp | 34sp | Screen titles |
| H2 | 700 | 22sp | 28sp | Section headers |
| H3 | 600 | 18sp | 24sp | Subsection headers |
| Body 1 | 400 | 16sp | 22sp | Primary body text |
| Body 2 | 400 | 14sp | 20sp | Secondary body text |
| Caption | 400 | 12sp | 16sp | Labels, timestamps |
| Button | 700 | 16sp | 20sp | CTA buttons |
| Overline | 800 | 10sp | 14sp | Badges, tags |

**Arabic (NotoSansArabic):**

Same scale with NotoSansArabic font family, with adjusted line heights for Arabic script (typically 1.4x font size vs. 1.25x for English).

### 2.3 Spacing System

| Token | Value | Usage |
|-------|-------|-------|
| `xs` | 4dp | Inline padding, icon gaps |
| `sm` | 8dp | Compact spacing, list item padding |
| `md` | 12dp | Standard element spacing |
| `lg` | 16dp | Screen edge padding, card internal padding |
| `xl` | 24dp | Section spacing |
| `2xl` | 32dp | Screen-level spacing |
| `3xl` | 48dp | Hero section spacing |

### 2.4 Corner Radius

| Token | Value | Usage |
|-------|-------|-------|
| `none` | 0dp | Full-bleed images |
| `sm` | 4dp | Tags, badges |
| `md` | 8dp | Buttons, input fields |
| `lg` | 12dp | Cards, bottom sheets |
| `xl` | 16dp | Modals, large cards |
| `full` | 50% | Avatar, circular buttons |

---

## 3. Core UI Components

### 3.1 Buttons

| Type | Visual | Usage |
|------|--------|-------|
| Primary CTA | Orange bg, white text, md radius | "Place Order", "Add to Cart" |
| Secondary CTA | White bg, orange border, orange text | "View Menu", "Change" |
| Tertiary | Text only, orange | "Skip", "Learn More" |
| Icon Button | Circular, icon only | Share, Favorite, Filter |
| Floating CTA | Fixed bottom, full width | "View Basket (2 items) - AED 85" |
| Pro CTA | Gold gradient bg | "Join Pro", "Start Free Trial" |

**CTA with Price (Feature: `ff_show_price_on_checkout_cta`):**
```
┌──────────────────────────────────────────┐
│  Place Order  ·  AED 82.25              │
└──────────────────────────────────────────┘
```

### 3.2 Cards

**Vendor Card:**
```
┌─────────────────────────────────────────┐
│  ┌────────┐                             │
│  │  Logo  │  Al Safadi                  │
│  │  80x80 │  Lebanese · Middle Eastern  │
│  └────────┘  ★ 4.5 (12.5K) · 30-45 min │
│              Free delivery with Pro 🏷️   │
│  ┌──────────────────────────────────┐   │
│  │  🏷️ 20% off  ·  Min AED 50      │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**Order Card (Home):**
```
┌─────────────────────────────────────────┐
│  🟢 Preparing  ·  Al Safadi             │
│  ETA: 25-35 min                         │
│  ┌──────────────────────────────────┐   │
│  │  Track Order    │    Need Help?   │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### 3.3 Bottom Sheets

Bottom sheets are heavily used for contextual actions:

| Bottom Sheet | Trigger | Content |
|-------------|---------|---------|
| Address selection | Tap delivery address | Saved addresses + map |
| Item detail | Tap menu item | Item image, description, options, add to cart |
| Item replacement | Out of stock during picking | Timer + replacement options |
| Voucher input | Tap voucher field | Voucher code entry + BIN validation |
| TGO promotion | Tap TGO badge | TGO benefits + opt-in |
| Pro subscription | Pro CTA | Plan selection + payment |
| Rider tip | After delivery | Preset amounts + custom |
| Cancellation survey | Cancel subscription | Reason selection + free text |
| Order modification | Modified order | Item changes + price difference |

### 3.4 Input Fields

| Type | Visual | Validation |
|------|--------|------------|
| Phone number | Country flag + dial code + number | E.164 format, `flutter_multi_formatter` |
| OTP | 6 individual digit boxes | Auto-fill via `sms_autofill` |
| Email | Standard text field with @ icon | RFC 5322 validation |
| Password | Obscured with visibility toggle | Min 8 chars, complexity |
| Address search | Search with map pin | Area validation |
| Card number | With card brand icon, formatted | Luhn check, brand detection |
| Voucher code | Uppercase, with apply button | Server-side validation |

---

## 4. Screen Specifications

### 4.1 Onboarding

```
┌──────────────────────────────────────────┐
│                                          │
│         [Illustration/Swipeable]         │
│                                          │
│    "Get access to AED 25 meals and       │
│     FREE delivery every day"             │
│                                          │
│    "Avoid the rush.                      │
│     Subscribe to smart lunch"            │
│                                          │
│    ○ ○ ● ○  (page indicators)           │
│                                          │
│    ┌──────────────────────────────┐      │
│    │      Get Started             │      │
│    └──────────────────────────────┘      │
│                                          │
│    Already have an account? Log in       │
│                                          │
└──────────────────────────────────────────┘
```

**Feature flag:** `exp_hide_skip_during_onboarding` can hide the skip button

### 4.2 Home Screen

```
┌──────────────────────────────────────────┐
│  📍 Delivering to Dubai Marina    ▼      │
│  ┌──────────────────────────────────┐    │
│  │ 🔍 Search for food or groceries │    │
│  └──────────────────────────────────┘    │
│                                          │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐    │
│  │Food│ │Groc│ │Phar│ │Dine│ │More│    │
│  └────┘ └────┘ └────┘ └────┘ └────┘    │
│                                          │
│  ── Active Order ──────────────────      │
│  ┌──────────────────────────────────┐    │
│  │ 🟢 Preparing · Al Safadi        │    │
│  │ ETA: 25-35 min                  │    │
│  │ [Track]            [Help]       │    │
│  └──────────────────────────────────┘    │
│                                          │
│  ── Pro Benefits ──────────────────      │
│  ┌──────────────────────────────────┐    │
│  │ 🏷️ Free delivery on Pro orders  │    │
│  │ [Explore Benefits]               │    │
│  └──────────────────────────────────┘    │
│                                          │
│  ── Offers ────────────────────────      │
│  ┌──────┐ ┌──────┐ ┌──────┐            │
│  │Vendor│ │Vendor│ │Vendor│  ← Swipe   │
│  │ Card │ │ Card │ │ Card │            │
│  └──────┘ └──────┘ └──────┘            │
│                                          │
│  ── Popular near you ──────────────      │
│  ┌──────────────────────────────────┐    │
│  │ Vendor list items...             │    │
│  └──────────────────────────────────┘    │
│                                          │
│  ══════════════════════════════════      │
│  🏠    🔍    📋    👤                    │
│  Home  Search Orders Profile             │
└──────────────────────────────────────────┘
```

**Feature flags:**
- `ff_homepage_incentive_raf_entry_point` — Refer-a-friend entry
- `ff_homepage_predict_and_feast_entrypoint` — Predict & Feast game
- `ff_homepage_wfp_swimlane` — WFP swimlane
- `ff_homepage_mm_adoption_enabled` — Market maturity adoption
- `exp_homepage_splash_overlay` — Splash overlay experiment

### 4.3 Vendor Menu Screen

```
┌──────────────────────────────────────────┐
│  ←  ┌────────────────────────────┐      │
│     │     Cover Image (200dp)     │      │
│     └────────────────────────────┘      │
│                                          │
│  Al Safadi                               │
│  Lebanese · Middle Eastern               │
│  ★ 4.5 (12.5K) · 30-45 min · AED 5     │
│                                          │
│  🏷️ Free delivery with Pro              │
│  🏷️ 20% off (Min AED 50)               │
│                                          │
│  ┌──────────────────────────────────┐    │
│  │ 🔍 Search menu                  │    │
│  └──────────────────────────────────┘    │
│                                          │
│  ── Starters ──────────────────────      │
│  ┌──────────────────────────────────┐    │
│  │ Hummus                    AED 18 │    │
│  │ Creamy chickpea dip...     [Add] │    │
│  └──────────────────────────────────┘    │
│  ┌──────────────────────────────────┐    │
│  │ Falafel Plate              AED 22│    │
│  │ Crispy chickpea fritters  [Add] │    │
│  └──────────────────────────────────┘    │
│                                          │
│  ── Main Course ───────────────────      │
│  ...                                     │
│                                          │
│  ┌──────────────────────────────────┐    │
│  │ View Basket (2) · AED 85.00  →  │    │
│  └──────────────────────────────────┘    │
└──────────────────────────────────────────┘
```

### 4.4 Order Tracking Screen

```
┌──────────────────────────────────────────┐
│  ← Order Tracking              Help →    │
│                                          │
│  ┌──────────────────────────────────┐    │
│  │                                  │    │
│  │         [Live Map]               │    │
│  │    🏍️ Rider icon moving          │    │
│  │    📍 Your location pin          │    │
│  │                                  │    │
│  └──────────────────────────────────┘    │
│                                          │
│  🟢 On the way                           │
│  Muhammad · Motorcycle · ★ 4.8           │
│  ETA: 12 minutes                         │
│                                          │
│  ┌─────────┐  ┌─────────┐  ┌────────┐   │
│  │ 📞 Call │  │ 💬 Chat │  │ 💰 Tip │   │
│  └─────────┘  └─────────┘  └────────┘   │
│                                          │
│  ── Order Details ────────────────       │
│  Al Safadi                               │
│  2x Hummus (Large)          AED 52.00    │
│  1x Mixed Grill             AED 65.00    │
│                                          │
│  Subtotal                   AED 117.00   │
│  Delivery fee                AED 5.00    │
│  Total                      AED 122.00   │
│                                          │
└──────────────────────────────────────────┘
```

---

## 5. Interaction Patterns

### 5.1 Add to Cart Animation

When adding an item to cart (feature: `ff_menu_item_details_micro_interactions_enabled`):

1. Item image scales down and animates toward the cart icon
2. Cart badge increments with a bounce animation
3. "Item successfully added to cart" accessibility announcement
4. Cart button at bottom updates total price

### 5.2 Pull-to-Refresh

All list screens support pull-to-refresh:
- Vendor listings refresh vendor data and availability
- Order tracking refreshes status and ETA
- Wallet refreshes balance and transactions

### 5.3 Swipe Gestures

| Gesture | Screen | Action |
|---------|--------|--------|
| Swipe right | Vendor card | Add to favorites |
| Swipe left | Cart item | Remove from cart |
| Swipe down | Any screen | Refresh content |
| Horizontal swipe | Home offers | Browse offer cards |

### 5.4 Long Press

| Target | Action |
|--------|--------|
| Vendor card | Quick preview (peek) |
| Menu item | Quick add without options |
| Address | Set as default |

---

## 6. Accessibility

### 6.1 Screen Reader Support

All interactive elements have meaningful accessibility labels (from `accessibility.*` translation keys):

| Element | Accessibility Label |
|---------|-------------------|
| Add to cart button | `"accessibility.cart.add_to_cart.action"` = "Add to cart" |
| Cart loading | `"accessibility.cart.add_to_cart.loading"` = "Adding to cart" |
| Cart success | `"accessibility.cart.add_to_cart.success"` = "Item successfully added to cart" |
| Cart error | `"accessibility.cart.add_to_cart.error"` = "Issue adding item to cart, try again" |
| Delivery address | `"accessibility.checkout.address.pre"` = "Delivering to" |
| Change address | `"accessibility.checkout.address.button"` = "Change address" |
| Suggested items | `"accessibility.cart.suggested_items.count"` = "Item {itemNumber} of {totalItemsCount}" |
| Credit card | `"accessibility.upw.credit_card"` = "Card ending in {digits}" |
| Tooltip | `"accessibility.shared.tooltip.hint"` = "Learn more" |

### 6.2 Accessibility Icon

The design system includes `ds_accessible` icon, indicating dedicated accessibility features for:
- Wheelchair-accessible restaurants (DineOut)
- Screen reader-optimized layouts
- High contrast mode support
- Font scaling support

### 6.3 Minimum Touch Target

All interactive elements meet the **48x48dp minimum touch target** guideline, with 8dp minimum spacing between adjacent targets.

---

## 7. Animation & Motion

### 7.1 Design System Animations

The `fluid` Flutter package provides reusable animation utilities:

| Animation | Duration | Easing | Usage |
|-----------|----------|--------|-------|
| Page transition | 300ms | Ease-out | Screen navigation |
| Bottom sheet | 250ms | Ease-out | Sheet show/hide |
| Card expansion | 200ms | Spring | Item detail expand |
| Button press | 100ms | Ease-in | CTA feedback |
| Shimmer | 1500ms | Linear loop | Loading placeholders |
| Sparkle | 500ms | Ease-out | GEM offer activation |

### 7.2 Micro-Interactions

Feature flag `ff_menu_item_details_micro_interactions_enabled` enables:

- Item image parallax on scroll
- Option selection bounce animation
- Quantity stepper smooth increment/decrement
- Add-to-cart fly animation

### 7.3 Lottie Animations

The `common_assets` package includes Lottie animations for:

- Order success celebration
- Pro subscription activation
- Reward points earning
- TGO badge pulse
- Loading states

---

## 8. Responsive Design

### 8.1 Device Class Breakpoints

| Class | Width | Target Devices |
|-------|-------|---------------|
| Compact | < 360dp | Small phones (Redmi, Samsung A series) |
| Medium | 360-412dp | Standard phones (iPhone, Pixel, Galaxy S) |
| Expanded | 412-600dp | Large phones, small tablets |
| Large | > 600dp | Tablets, foldable devices |

### 8.2 Foldable Device Support

The feature flag `ff_foldable_device_tracking` enables responsive layout adaptation for foldable devices:

- **Folded**: Standard phone layout
- **Unfolded**: Two-pane layout (list + detail)
- **Tabletop**: Split-screen for video + ordering

### 8.3 Landscape Mode

The app supports landscape orientation for:

- Map-based screens (order tracking, address selection)
- DineOut map view
- Video content (vendor stories)

---

## 9. Loading States

### 9.1 Skeleton/Shimmer Loading

All data-driven screens display skeleton loading states:

| Screen | Skeleton Pattern |
|--------|-----------------|
| Home | Card placeholders with shimmer |
| Vendor listing | Card grid with shimmer |
| Menu | Category headers + item rows with shimmer |
| Order tracking | Map placeholder + status skeleton |
| Search results | Card list with shimmer |

### 9.2 Empty States

| Screen | Title | Description | CTA |
|--------|-------|-------------|-----|
| Search (no results) | "No results for {query}" | "Look out for sneaky typos, or try a different word" | — |
| Grocery search (no results) | "We couldn't find {query}" | "Check the spelling or try another search" | — |
| Order history (empty) | "No orders yet" | "Your order history will appear here" | "Start ordering" |
| Saved addresses (empty) | "No saved addresses" | "Add a delivery address to get started" | "Add address" |
| Wallet (no transactions) | "No transactions" | "Your transaction history will appear here" | — |

### 9.3 Error States

| Type | Title | Description | Action |
|------|-------|-------------|--------|
| Network error | "Something went wrong" | "Check your connection and try again" | "Retry" |
| Cached results | "Some restaurants aren't loading" | "Looks like you're offline, but here are some picks which you can still explore" | — |
| Server error | "Something went wrong" | "We're working on fixing it" | "Try again" |
| Cart add failure | — | "Issue adding item to cart, try again" | — |
