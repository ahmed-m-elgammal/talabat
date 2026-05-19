# 03 — UI/UX Scope

## 1. Overview

This plan defines the UI/UX implementation scope for the Talabat-like MVP, based on the production design specifications in `docs/13_ui_ux_specifications.md`. The MVP implements the MM3 design system with brand colors, typography, and component patterns adapted for a single-country, food-delivery-focused product.

The design must support **English (LTR) and Arabic (RTL)** layouts, provide **offline-first visual feedback**, and deliver a **fast, task-completion-oriented experience** optimized for mobile devices.

---

## 2. Design System Foundation (MVP)

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

### 2.2 Typography

**English**: TTCommonsPro (400, 600, 700, 800)

| Style | Weight | Size | Usage |
|-------|--------|------|-------|
| H1 | 800 | 28sp | Screen titles |
| H2 | 700 | 22sp | Section headers |
| H3 | 600 | 18sp | Subsection headers |
| Body 1 | 400 | 16sp | Primary body text |
| Body 2 | 400 | 14sp | Secondary body text |
| Caption | 400 | 12sp | Labels, timestamps |
| Button | 700 | 16sp | CTA buttons |

**Arabic**: NotoSansArabic (400, 500, 700, 800) — Same scale with adjusted line heights (1.4x font size).

### 2.3 Spacing & Layout Tokens

| Token | Value | Usage |
|-------|-------|-------|
| xs | 4dp | Inline padding, icon gaps |
| sm | 8dp | Compact spacing |
| md | 12dp | Standard element spacing |
| lg | 16dp | Screen edge padding |
| xl | 24dp | Section spacing |
| 2xl | 32dp | Screen-level spacing |

### 2.4 Corner Radius

| Token | Value | Usage |
|-------|-------|-------|
| sm | 4dp | Tags, badges |
| md | 8dp | Buttons, input fields |
| lg | 12dp | Cards, bottom sheets |
| xl | 16dp | Modals |
| full | 50% | Avatar, circular buttons |

---

## 3. Core UI Components (MVP)

### 3.1 Buttons

| Component | Visual Spec | Usage |
|-----------|-------------|-------|
| Primary CTA | Orange bg (#FF5A00), white text, md radius (8dp), min height 48dp | "Place Order", "Add to Cart" |
| Secondary CTA | White bg, orange border, orange text, md radius | "View Menu", "Change" |
| Tertiary | Text only, orange | "Skip", "Learn More" |
| Icon Button | Circular (48dp), icon only | Share, Favorite, Filter |
| Floating CTA | Fixed bottom, full width, shadow elevation | "View Basket (2) - AED 85" |

### 3.2 Cards

| Component | Content |
|-----------|---------|
| Vendor Card | Logo (80x80), name, cuisine, rating, delivery time, delivery fee, offer badge, Pro badge |
| Order Card (Home) | Status icon, vendor name, ETA, Track/Help buttons |
| Menu Item Row | Name, description (2-line max), price, Add button / quantity stepper |
| Order History Card | Vendor name, items summary, total, date, status badge, Reorder button |

### 3.3 Bottom Sheets

| Sheet | Trigger | Content |
|-------|---------|---------|
| Address selection | Tap delivery address | Saved addresses + map picker |
| Item detail | Tap menu item | Image, description, options, quantity, Add to Cart |
| Sort & Filter | Tap filter bar | Sort options, cuisine filters, delivery filters |
| Voucher input | Tap voucher field in cart | Voucher code entry + apply |
| Cancellation | Cancel order | Reason selection + confirm |

### 3.4 Input Fields

| Type | Spec |
|------|------|
| Phone number | Country flag + dial code + number, E.164 validation |
| OTP | 6 individual digit boxes, auto-fill, cursor animation |
| Search | Search icon, clear button, placeholder per vertical |
| Card number | Brand icon, formatted groups, Luhn check |
| Address | Search with map, area validation |

---

## 4. Screen Specifications (MVP)

### 4.1 Screen Inventory

| # | Screen | Route | Priority | Key Elements |
|---|--------|-------|----------|-------------|
| S1 | Splash / Onboarding | `/` | P0 | App logo, language selector, get started |
| S2 | Login | `/auth/login` | P0 | Phone input, social login, skip |
| S3 | OTP Verification | `/auth/otp` | P0 | 6-digit input, resend timer, verify |
| S4 | Registration | `/auth/signup` | P0 | Name, email, phone, password, consent |
| S5 | Home | `/` | P0 | Location, search, vertical tabs, vendor list, active order |
| S6 | Search | `/search` | P0 | Search bar, autocomplete, results, filters |
| S7 | Vendor Detail | `/vendor/{id}` | P0 | Cover, info, menu categories, items, cart FAB |
| S8 | Cart | `/cart` | P0 | Items, quantity, voucher, total, checkout CTA |
| S9 | Checkout | `/checkout` | P0 | Address, time, payment, summary, place order |
| S10 | Add Card | `/payment/add-card` | P0 | Card number, expiry, CVV, 3DS WebView |
| S11 | Order History | `/orders` | P0 | Order list, reorder |
| S12 | Order Detail | `/orders/{id}` | P0 | Status timeline, items, pricing, rider info |
| S13 | Order Tracking | `/orders/{id}/tracking` | P0 | Live map, rider position, ETA, actions |
| S14 | Profile | `/profile` | P0 | User info, addresses, payment, settings |
| S15 | Address Manager | `/profile/addresses` | P0 | Saved addresses, add/edit/delete |
| S16 | Rider Chat | `/orders/{id}/chat` | P1 | Message list, input, quick messages |
| S17 | Favorites | `/favorites` | P1 | Favorited vendors list |

---

## 5. Tasks

### 5.1 Design System Tasks

---

#### Task UX-DS-01: Design System Package — Foundation
**Description**: Create the `design_system` Flutter package with color constants, typography definitions, spacing tokens, and corner radius tokens. Implement `TalabatTheme` class with `lightTheme` and `darkTheme` configurations.
**Dependencies**: None
**Acceptance Criteria**:
- All brand colors defined as `Color` constants
- Typography scale implemented with `TextStyle` for each style
- Spacing and radius tokens as `double` constants
- `TalabatTheme.lightTheme` returns `ThemeData` with all specifications
- `TalabatTheme.darkTheme` returns `ThemeData` for dark mode
- Arabic font (NotoSansArabic) loaded and configured for RTL text
- Theme switches at runtime without app restart

---

#### Task UX-DS-02: Button Components
**Description**: Implement all button variants: PrimaryCta, SecondaryCta, TertiaryButton, IconButton, FloatingCta. Support loading state, disabled state, and accessibility labels.
**Dependencies**: UX-DS-01
**Acceptance Criteria**:
- Primary CTA: orange bg, white text, 8dp radius, 48dp min height
- Loading state: spinner replaces text
- Disabled state: gray bg, no tap response
- Floating CTA: fixed position, shadow, full width
- All buttons meet 48x48dp minimum touch target
- Accessibility: semantic label announced by screen reader
- RTL: icon positions flip in Arabic

---

#### Task UX-DS-03: Card Components
**Description**: Implement VendorCard, OrderCard, MenuItemRow, and OrderHistoryCard components. Support loading skeleton state and error state.
**Dependencies**: UX-DS-01
**Acceptance Criteria**:
- VendorCard: logo, name, cuisine, rating, delivery time, fee, offer badge
- Skeleton state: shimmer placeholder matching card layout
- OrderCard: status icon with color, vendor, ETA, action buttons
- MenuItemRow: name (1 line), description (2 lines max), price, Add/stepper
- OrderHistoryCard: vendor, items count, total, status badge, date, reorder
- All cards: 12dp corner radius, lg internal padding
- Tap targets meet 48dp minimum

---

#### Task UX-DS-04: Bottom Sheet Component
**Description**: Implement reusable bottom sheet component with drag handle, scrollable content area, and dismiss behavior. Support multiple content types.
**Dependencies**: UX-DS-01
**Acceptance Criteria**:
- Drag handle at top (centered, 40dp wide, 4dp height)
- Sheet animates up from bottom (250ms ease-out)
- Scrollable content area
- Dismiss: swipe down or tap outside
- Sheet covers 60-90% of screen height based on content
- No gap between sheet and keyboard when input focused

---

#### Task UX-DS-05: Input Field Components
**Description**: Implement typed input fields: PhoneInput, OtpInput, SearchInput, CardInput, AddressInput. All with validation, error messages, and accessibility.
**Dependencies**: UX-DS-01
**Acceptance Criteria**:
- PhoneInput: country flag + dial code + number, E.164 formatting
- OtpInput: 6 boxes, auto-focus next on entry, auto-submit on complete
- SearchInput: search icon, clear button, placeholder text
- CardInput: brand icon detection, grouped formatting, Luhn check
- Error state: red border + helper text below
- All inputs: 48dp min height, 8dp corner radius, lg horizontal padding
- RTL: text alignment flips for Arabic

---

### 5.2 Screen Implementation Tasks

---

#### Task UX-SCR-01: Splash & Onboarding
**Description**: Implement splash screen with app logo animation (1.5s), followed by onboarding carousel (3 slides) with swipeable illustrations and "Get Started" CTA.
**Dependencies**: UX-DS-02
**Acceptance Criteria**:
- Splash: logo animation, auto-advance after 1.5s
- Onboarding: 3 swipeable cards with illustrations and text
- Page indicators (dots) at bottom
- "Get Started" primary CTA
- "Already have an account? Log in" link
- "Skip" button (top right)
- Remembers if onboarding completed (don't show again)

---

#### Task UX-SCR-02: Login Screen
**Description**: Implement login screen with phone number input, social login buttons (Google, Apple), and guest mode option.
**Dependencies**: UX-DS-05, F-AUTH-01
**Acceptance Criteria**:
- App logo at top
- Phone number input with country flag/dial code
- "Continue" primary CTA
- Divider: "or"
- Social login buttons: Google, Apple (with correct branding)
- "Skip" text button for guest mode
- All text available in English + Arabic
- Keyboard auto-opens on phone input focus

---

#### Task UX-SCR-03: OTP Verification Screen
**Description**: Implement OTP screen with 6-digit boxes, resend timer, and verification logic.
**Dependencies**: UX-DS-05, F-AUTH-01
**Acceptance Criteria**:
- Title: "Verify your number"
- Subtitle: "We sent a code to +971 ****567"
- 6 digit boxes with auto-advance focus
- Auto-fill via SMS (sms_autofill plugin)
- Resend button with 60-second countdown
- Error: "Invalid code" with retry
- Success: auto-navigate to home
- Accessibility: each box announced as "digit X of 6"

---

#### Task UX-SCR-04: Home Screen
**Description**: Implement the full home screen layout as specified in `docs/13_ui_ux_specifications.md` Section 4.2.
**Dependencies**: UX-DS-03, F-HOME-01
**Acceptance Criteria**:
- Location bar: "Delivering to {area}" with chevron
- Search bar with placeholder
- Vertical tab: Food
- Active order card (conditional: only if active order exists)
- Offers horizontal carousel
- "Popular near you" vendor list
- Bottom navigation: Home (active), Search, Orders, Profile
- Pull-to-refresh with custom indicator
- Shimmer skeleton for loading state
- Offline banner: "You're offline" with retry

---

#### Task UX-SCR-05: Search Screen
**Description**: Implement search screen with autocomplete, results, and filtering as specified in `docs/16_search_discovery_system.md`.
**Dependencies**: UX-DS-05, F-SEARCH-01
**Acceptance Criteria**:
- Auto-focus search bar on entry
- Autocomplete dropdown with recent searches and suggestions
- Tab bar: All, Food
- Vendor results section with cards
- Dish/item results section
- Sort & Filter bottom sheet
- Active filter chips
- Empty state: "No results for {query}" with suggestions
- Cached results shown offline with toast

---

#### Task UX-SCR-06: Vendor Detail & Menu
**Description**: Implement vendor detail screen with menu as specified in `docs/13_ui_ux_specifications.md` Section 4.3.
**Dependencies**: UX-DS-03, F-VENDOR-01
**Acceptance Criteria**:
- Cover image with gradient overlay, back button, favorite button
- Vendor info: name, cuisine, rating, delivery time, fee
- Offer badges row
- Menu search bar (within vendor)
- Category headers (sticky on scroll)
- Menu item rows with Add button
- Floating cart button: "View Basket ({n}) - {price}"
- Item tap → item detail bottom sheet
- Scroll to category from category chips at top

---

#### Task UX-SCR-07: Cart & Checkout Screens
**Description**: Implement cart screen and multi-step checkout flow.
**Dependencies**: UX-DS-02, UX-DS-03, F-CART-01, F-CHECKOUT-01
**Acceptance Criteria**:
- Cart: item list with stepper, swipe-delete, voucher field, total, CTA
- Cross-vendor dialog: "Clear cart?" bottom sheet
- Checkout step 1: Address (pre-selected, change button)
- Checkout step 2: Delivery time (ASAP)
- Checkout step 3: Payment method selection
- Order summary with item count and pricing
- "Place Order - {total}" CTA
- Loading overlay during order placement
- Success → order tracking screen
- Error → specific error message with retry

---

#### Task UX-SCR-08: Order Tracking Screen
**Description**: Implement order tracking screen with live map as specified in `docs/13_ui_ux_specifications.md` Section 4.4.
**Dependencies**: F-ORDER-03, F-CORE-06
**Acceptance Criteria**:
- Full map view with: vendor marker, rider marker, delivery pin
- Rider icon moves in real-time
- Status bar: "On the way" with color indicator
- Rider info: name, vehicle, rating
- ETA: "12 minutes" (updates dynamically)
- Action row: Call, Chat, Tip buttons
- Expandable order details section below map
- Map launches in full landscape on rotation
- Rider chat accessible from action row (P1)
- Offline: last known position with reconnect indicator

---

#### Task UX-SCR-09: Order History Screen
**Description**: Implement order history list with reorder functionality.
**Dependencies**: UX-DS-03, F-ORDER-01
**Acceptance Criteria**:
- Active orders section at top (if any)
- Past orders: vendor name, items count, total, date, status
- Reorder button on past orders
- Pull-to-refresh
- Empty state: "No orders yet" with "Start ordering" CTA
- Pagination: load more on scroll to bottom

---

#### Task UX-SCR-10: Profile & Settings
**Description**: Implement profile screen with user info, addresses, payment methods, and settings.
**Dependencies**: UX-DS-03, F-PROFILE-01
**Acceptance Criteria**:
- User avatar/name at top
- Account info section: name, email, phone (tappable to edit)
- Addresses section: list with add/edit/delete
- Payment methods section: list with add/remove
- Settings: language, dark mode, notifications
- Logout button
- About & legal links

---

### 5.3 Accessibility Tasks

---

#### Task UX-A11Y-01: Screen Reader Support
**Description**: Add semantic labels to all interactive elements. Follow accessibility patterns from `docs/13_ui_ux_specifications.md` Section 6.
**Dependencies**: All screen tasks
**Acceptance Criteria**:
- All buttons have meaningful accessibility labels
- Cart add: "Add to cart" → "Adding to cart" → "Item successfully added to cart" / "Issue adding item to cart"
- Checkout address: "Delivering to {area}" + "Change address" button
- Suggested items: "Item {n} of {total}"
- OTP: "Digit {n} of 6"
- Order status: "Order status: {status}"
- Rider location: "Rider is {distance} away, ETA {minutes} minutes"

---

#### Task UX-A11Y-02: RTL Layout Support
**Description**: Ensure all screens render correctly in RTL (Arabic) mode. This includes text direction, icon mirroring, layout flipping, and navigation patterns.
**Dependencies**: All screen tasks
**Acceptance Criteria**:
- All text renders right-to-left in Arabic
- Navigation back arrow points right in RTL
- Icons that imply direction are mirrored (e.g., back, forward)
- Padding/margin symmetric (using `EdgeInsetsDirectional`)
- Swipe gestures respect text direction
- Currency formatting: "AED 85" in LTR, "85 AED" in RTL
- Mixed Arabic/English text renders correctly (bidirectional)

---

#### Task UX-A11Y-03: Loading & Empty States
**Description**: Implement shimmer loading skeletons for all data-driven screens and empty states for all list views. Follow `docs/13_ui_ux_specifications.md` Sections 9.1 and 9.2.
**Dependencies**: All screen tasks
**Acceptance Criteria**:
- Home: shimmer on vendor cards, offer cards
- Menu: shimmer on category headers and item rows
- Search: shimmer on result cards
- Order tracking: map placeholder + status skeleton
- Empty states for: search (no results), order history (no orders), cart (empty)
- Error states for: network error, server error, cached results

---

## 6. Animation Specifications

| Animation | Duration | Easing | Trigger |
|-----------|----------|--------|---------|
| Page transition | 300ms | Ease-out | Screen navigation |
| Bottom sheet | 250ms | Ease-out | Sheet show/hide |
| Button press | 100ms | Ease-in | CTA tap |
| Shimmer loading | 1500ms | Linear loop | Data loading |
| Cart badge bounce | 200ms | Spring | Item added to cart |
| Pull-to-refresh | 300ms | Ease-out | Pull gesture |
| Status change | 300ms | Ease-out | Order status update |

---

## 7. Responsive Breakpoints (MVP)

| Class | Width | Target |
|-------|-------|--------|
| Compact | < 360dp | Small phones |
| Medium | 360-412dp | Standard phones |
| Expanded | > 412dp | Large phones |

MVP focuses on Compact and Medium. Expanded support is nice-to-have. No tablet/foldable layout for MVP.

---

## 8. Dark Mode

Dark mode follows the production app pattern:

| Element | Light Mode | Dark Mode |
|---------|-----------|-----------|
| Background | `#FFFFFF` | `#121212` |
| Surface | `#F3F4F6` | `#1E1E1E` |
| Primary text | `#1A1A2E` | `#FFFFFF` |
| Secondary text | `#6B7280` | `#9CA3AF` |
| CTA | `#FF5A00` | `#FF5A00` (unchanged) |
| Card | `#FFFFFF` | `#1E1E1E` |
| Error | `#EF4444` | `#EF4444` (unchanged) |

Dark mode is toggle-only in MVP (no automatic system-following for simplicity).
