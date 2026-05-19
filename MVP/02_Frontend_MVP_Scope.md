# 02 — Frontend MVP Scope

## Overview

The MVP frontend is a **Flutter** mobile application targeting iOS and Android, following the same architecture described in `docs/12_frontend_architecture.md`: feature-first modular design with BLoC/Cubit state management, clean architecture (presentation → domain → data layers), and a shared design system package. The MVP focuses on the core ordering flow: browse → select → cart → checkout → track.

The full Talabat app supports 20+ feature modules, dual platform services (Google + HMS), 4 languages (English, Arabic, Arabic dialects, Kurdish), and 60+ native plugins. The MVP strips this to 8 core feature modules, Google Play Services only, English + Arabic, and ~15 essential plugins.

---

## Feature Modules

### T2.1 — Auth Feature Module

**Description:** Implements the authentication flow: mobile OTP login, guest mode, and basic registration. Supports the progressive auth pattern where users can browse without logging in but must authenticate to place orders. Uses `sms_autofill` for OTP auto-fill on Android.

**Dependencies:** Backend T1.1 (User Service), T1.6 (API Gateway).

**Acceptance Criteria:**
- [ ] Login screen with phone number input (E.164 format, country flag + dial code)
- [ ] OTP verification screen with 6-digit boxes and auto-fill via SMS
- [ ] Guest mode: skip button on login screen, browse freely without auth
- [ ] Auth guard: redirect to login when user tries to place order without auth
- [ ] JWT token storage in FlutterSharedPreferences with secure storage for refresh token
- [ ] Auto token refresh on 401 responses
- [ ] Basic registration: first name, last name, email (optional), DOB
- [ ] Logout clears tokens and navigates to home

**Phase:** MVP

---

### T2.2 — Home Feature Module

**Description:** The main landing screen showing delivery address selector, vertical tabs (food only for MVP), vendor listing with cards, and an active order card if applicable. Implements pull-to-refresh and skeleton loading states.

**Dependencies:** T2.1 (auth for address management), Backend T1.2 (vendor listing).

**Acceptance Criteria:**
- [ ] Delivery address selector at top with dropdown showing saved addresses
- [ ] Single vertical tab: "Food" (other tabs grayed out with "Coming soon")
- [ ] Vendor listing as scrollable list of vendor cards (logo, name, cuisine, rating, delivery time, delivery fee)
- [ ] Active order card at top when user has an in-progress order (status + ETA + Track/Help buttons)
- [ ] Pull-to-refresh reloads vendor listing
- [ ] Skeleton/shimmer loading state while fetching vendors
- [ ] Empty state when no vendors deliver to selected address
- [ ] Offline banner when network is unavailable

**Phase:** MVP

---

### T2.3 — Vendor & Menu Feature Module

**Description:** The vendor detail screen showing cover image, vendor info (name, cuisine, rating, delivery time, delivery fee), and the full menu organized by categories. Supports adding items to cart with option selection via bottom sheet. Search within menu.

**Dependencies:** T2.2 (navigation from home), Backend T1.2 (vendor menu API).

**Acceptance Criteria:**
- [ ] Vendor header: cover image, name, cuisine tags, rating (avg + count), delivery time range, delivery fee
- [ ] Menu categories as horizontal scrollable tabs, scrolling to category on tap
- [ ] Menu items: name, description, price, "Add" button; tap opens item detail bottom sheet
- [ ] Item detail bottom sheet: image, description, required/optional option selection, quantity stepper, special instructions field, total price calculation, "Add to Cart" button
- [ ] Sticky cart button at bottom: "View Basket (N items) · {total price}"
- [ ] Menu search: filter items by name within current vendor
- [ ] Unavailable items shown greyed with "Unavailable" label
- [ ] Menu caching with ETag: serve from cache if unchanged (304 response)

**Phase:** MVP

---

### T2.4 — Cart Feature Module

**Description:** Shopping cart management with item list, quantity adjustment, item removal, voucher application, and price breakdown. Single-vendor constraint enforced.

**Dependencies:** T2.3 (items added from menu), Backend T1.3 (cart APIs).

**Acceptance Criteria:**
- [ ] Cart screen shows items from current vendor with quantity steppers (+/-)
- [ ] Swipe-to-delete on cart items with confirmation
- [ ] Voucher code input field with "Apply" button; server-side validation with error display
- [ ] Price breakdown: subtotal, delivery fee, service fee, discount, total
- [ ] "Clear cart" confirmation when switching vendors
- [ ] Cart persists across app restarts (server-side cache + local SQLite)
- [ ] Empty cart state: illustration + "Start ordering" CTA
- [ ] Checkout CTA at bottom with total amount

**Phase:** MVP

---

### T2.5 — Checkout Feature Module

**Description:** The checkout flow: address selection → payment method selection → order confirmation. Streamlined from Talabat's 5-step checkout to 3 steps for MVP.

**Dependencies:** T2.4 (cart data), T2.1 (auth), Backend T1.3 (order), T1.4 (payment).

**Acceptance Criteria:**
- [ ] Step 1: Address selection — default address pre-selected, option to change from saved addresses
- [ ] Step 2: Payment method — saved cards (if any), add new card, cash on delivery
- [ ] Step 3: Order confirmation — final summary with all items, pricing, address, payment method, "Place Order" CTA
- [ ] Card entry via Stripe/Checkout.com SDK with client-side tokenization
- [ ] Payment processing indicator (spinner) during authorization
- [ ] Order success screen with order number and "Track Order" CTA
- [ ] Error handling: payment declined → suggest alternative method; network error → retry option

**Phase:** MVP

---

### T2.6 — Order Tracking Feature Module

**Description:** Real-time order tracking with map showing rider position, order status timeline, rider info, and action buttons (call, chat). Uses Firebase RTDB for live rider location updates.

**Dependencies:** T2.5 (order placed), Backend T1.5 (dispatch), T1.3 (order status), T5.1 (Firebase RTDB integration).

**Acceptance Criteria:**
- [ ] Order tracking screen with map showing vendor location, rider position, and delivery address
- [ ] Rider marker moves in real-time as GPS updates arrive (every 5–10 seconds)
- [ ] Status timeline: Ordered → Preparing → On the way → Delivered (current step highlighted)
- [ ] Rider info card: name, vehicle type, rating
- [ ] Action buttons: Call rider (opens phone dialer), Chat (opens chat screen)
- [ ] ETA display updated in real-time
- [ ] "Order Delivered" confirmation with tip prompt (Phase 2: actual tip flow; MVP: dismiss only)
- [ ] Pull-to-refresh for status updates as fallback

**Phase:** MVP

---

### T2.7 — Order History Feature Module

**Description:** Paginated list of past orders with reorder capability. Each order card shows vendor name, items summary, total, status, and date.

**Dependencies:** T2.1 (auth required), Backend T1.3 (order history API).

**Acceptance Criteria:**
- [ ] Order history list with infinite scroll pagination
- [ ] Order cards: vendor name + logo, item count, total amount, status badge, date
- [ ] Tap order card → order detail screen with full item list, pricing, delivery info
- [ ] "Reorder" button on order detail: adds available items to cart, shows unavailable items notice
- [ ] Empty state: "No orders yet" with "Start ordering" CTA
- [ ] Filter tabs: All, Active, Completed, Cancelled

**Phase:** MVP

---

### T2.8 — Profile & Settings Feature Module

**Description:** User profile management, address book, saved payment methods, notification preferences, and app settings (language, theme).

**Dependencies:** T2.1 (auth), Backend T1.1 (profile APIs), T1.4 (payment methods).

**Acceptance Criteria:**
- [ ] Profile screen: name, email, phone, DOB — all editable
- [ ] Address book: list of saved addresses, add/edit/delete, set default
- [ ] Payment methods: list of saved cards (last 4 digits, brand, expiry), add new card, delete
- [ ] Settings: language toggle (English/Arabic), push notification toggle
- [ ] Logout button
- [ ] Account deletion request (per app store requirements)

**Phase:** MVP

---

## Infrastructure Tasks

### T2.9 — Design System Package

**Description:** Creates the shared `design_system` Flutter package with the MM3-inspired theme: brand colors (orange primary), typography scale, spacing system, corner radius tokens, and reusable UI components (buttons, cards, input fields, bottom sheets). Supports light and dark themes with RTL layout for Arabic.

**Dependencies:** None (foundational).

**Acceptance Criteria:**
- [ ] Color tokens: primary orange (#FF5A00), dark (#1A1A2E), gray, success, warning, error
- [ ] Typography scale: Display, H1, H2, H3, Body1, Body2, Caption, Button, Overline
- [ ] Spacing tokens: xs(4), sm(8), md(12), lg(16), xl(24), 2xl(32), 3xl(48)
- [ ] Corner radius tokens: none(0), sm(4), md(8), lg(12), xl(16), full(50%)
- [ ] Reusable components: PrimaryButton, SecondaryButton, VendorCard, CartItemCard, InputField, SearchBar, BottomSheet
- [ ] Light and dark theme support
- [ ] RTL layout support for Arabic (directional spacing, mirrored icons)
- [ ] Minimum touch target 48x48dp for all interactive elements

**Phase:** MVP

---

### T2.10 — Navigation & Routing

**Description:** Sets up declarative routing using `go_router` with deep link support. Defines all route paths matching the full architecture's navigation structure (simplified for MVP routes).

**Dependencies:** T2.9 (design system for page transitions).

**Acceptance Criteria:**
- [ ] Route definitions: /, /search, /vendor/{id}, /cart, /checkout, /orders, /orders/{id}, /orders/{id}/tracking, /profile, /settings, /auth/login
- [ ] Deep link support: `talabat://` scheme for order tracking, vendor pages
- [ ] Auth guard: redirects unauthenticated users to login for protected routes
- [ ] Page transition animations (300ms ease-out)
- [ ] Back navigation works correctly on all screens

**Phase:** MVP

---

### T2.11 — Localization (English + Arabic)

**Description:** Implements bilingual support with English (LTR) and Arabic (RTL). Uses Flutter's `l10n` package with ARB files. All user-facing strings are externalized.

**Dependencies:** T2.9 (design system RTL support).

**Acceptance Criteria:**
- [ ] All UI strings in `app_en.arb` and `app_ar.arb` files
- [ ] Language switching in settings without app restart
- [ ] RTL layout flips correctly: text alignment, icons, navigation direction
- [ ] Bidirectional text support (Arabic + English mixed content)
- [ ] Currency formatting: AED for MVP (extensible to other currencies)
- [ ] Number formatting respects locale (Arabic-Indic numerals in Arabic mode)

**Phase:** MVP

---

### T2.12 — Network & Offline Layer

**Description:** Implements the HTTP client with interceptors for auth token injection, refresh token handling, and error mapping. Adds offline support with cached data for vendor listings and menus when network is unavailable.

**Dependencies:** T2.1 (auth tokens for interceptor).

**Acceptance Criteria:**
- [ ] HTTP client with base URL configuration (per environment: dev, staging, production)
- [ ] Auth interceptor adds Bearer token to all requests; handles 401 with token refresh
- [ ] Error mapping: HTTP status codes → user-friendly error messages
- [ ] Connectivity monitoring: shows offline banner when network is lost
- [ ] Cached vendor listings served when offline with "You're offline" notice
- [ ] Cart operations queued locally when offline, replayed on reconnection

**Phase:** MVP

---

## Phase 2 Frontend Tasks (Not for MVP)

### T2.P2.1 — Q-Commerce (Grocery) Feature Module
**Description:** Grocery vertical with finite stock items, category browsing, shopping lists, and item replacement flow. Requires backend inventory service.
**Phase:** Phase 2

### T2.P2.2 — Pharmacy Feature Module
**Description:** Pharmacy vertical with prescription upload, insurance integration, and age-restricted items.
**Phase:** Phase 2

### T2.P2.3 — DineOut Feature Module
**Description:** Restaurant reservations with capacity-based availability, BOGO packages, and bill payment.
**Phase:** Phase 2

### T2.P2.4 — Wallet & BNPL Feature Modules
**Description:** talabat Pay wallet (balance, top-up, transactions) and PostPaid BNPL (installment dashboard, payment, rewind).
**Phase:** Phase 2

### T2.P2.5 — Subscription (Pro) Feature Module
**Description:** Pro subscription management, plan selection, free delivery benefits, family plans.
**Phase:** Phase 2

### T2.P2.6 — Rewards Feature Module
**Description:** Loyalty points balance, earn/spend history, burn options (free delivery, money off, charity).
**Phase:** Phase 2

### T2.P2.7 — AI Chat Feature Module
**Description:** ChatGPT integration for conversational food discovery and order support.
**Phase:** Phase 2

### T2.P2.8 — HMS (Huawei Mobile Services) Support
**Description:** Dual platform services for HMS devices: Huawei Maps, HMS Push Kit, Huawei ID auth.
**Phase:** Phase 2

### T2.P2.9 — Performance Optimization
**Description:** Baseline profiles, deferred component loading, image caching optimization, HTTP/2 experimentation.
**Phase:** Phase 2

### T2.P2.10 — Advanced Search with Elasticsearch
**Description:** Full-text search with Arabic tokenization, autocomplete, multi-search, photo-to-list, and sponsored content.
**Phase:** Phase 2
