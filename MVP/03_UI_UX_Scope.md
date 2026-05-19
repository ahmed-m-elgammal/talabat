# 03 — UI/UX Scope

## Overview

The MVP UI/UX follows Talabat's MM3 design system (see `docs/13_ui_ux_specifications.md`) with simplified scope. We implement the core ordering flow screens with the brand's visual language: orange primary (#FF5A00), clean card-based layouts, bottom sheet interactions, and RTL support for Arabic. The MVP focuses on task completion speed — users should be able to place an order with minimal taps.

This document specifies which screens, components, and interactions are included in MVP vs. deferred to Phase 2.

---

## MVP Screens

### T3.1 — Onboarding Screen

**Description:** A lightweight onboarding with 2–3 swipeable pages introducing the app's value proposition, followed by a "Get Started" CTA and "Already have an account? Log in" link. Skip button allows bypassing onboarding entirely.

**Dependencies:** T2.9 (design system), T2.11 (localization).

**Acceptance Criteria:**
- [ ] 2–3 swipeable pages with illustration, headline, and description text
- [ ] Page indicator dots showing current position
- [ ] "Get Started" primary CTA button
- [ ] "Already have an account? Log in" text link
- [ ] Skip button in top-right corner (can be hidden via feature flag)
- [ ] Onboarding completion state saved locally (SharedPreferences)
- [ ] RTL layout for Arabic: swipe direction reversed, text aligned right

**Phase:** MVP

---

### T3.2 — Login Screen

**Description:** Phone number input with OTP verification. Minimal friction: enter phone, receive OTP, auto-verify, done. Guest mode "Skip" option available.

**Dependencies:** T2.1 (auth module), T2.9 (design system).

**Acceptance Criteria:**
- [ ] Phone number input with country flag + dial code selector (single country for MVP)
- [ ] "Continue" button sends OTP
- [ ] OTP screen with 6 individual digit boxes, auto-fill via SMS
- [ ] Resend OTP link with countdown timer (60 seconds)
- [ ] Error states: invalid phone format, too many OTP requests, wrong OTP
- [ ] Guest mode "Skip" link at bottom
- [ ] Social login buttons (Google, Apple) for future use — grayed out with "Coming soon" for MVP

**Phase:** MVP

---

### T3.3 — Home Screen

**Description:** The primary screen users see after login. Contains delivery address selector, search bar, food vertical tab, and vendor listing. Active order card appears at top when applicable.

**Dependencies:** T2.2 (home module), T2.9 (design system).

**Acceptance Criteria:**
- [ ] Top bar: delivery address with dropdown arrow, tapping opens address selector
- [ ] Search bar below address: placeholder "Search for food or groceries" (tapping navigates to search screen)
- [ ] Vertical tabs row: "Food" active, "Grocery"/"Pharmacy"/"DineOut" tabs with "Coming soon" label
- [ ] Active order card (if applicable): vendor name, status badge (colored dot + text), ETA, Track and Help buttons
- [ ] Vendor listing: scrollable list of vendor cards
- [ ] Vendor card: logo (80x80), name, cuisine types, rating (★ avg + count), delivery time range, delivery fee, "Free delivery with Pro" label (Phase 2; for MVP, show delivery fee)
- [ ] Offer badge on vendor card if applicable (e.g., "20% off · Min AED 50")
- [ ] Bottom navigation bar: Home, Search, Orders, Profile (icons + labels)
- [ ] Skeleton/shimmer loading state for vendor cards
- [ ] Pull-to-refresh gesture reloads data
- [ ] Offline banner: "Looks like you're offline, but here are some picks which you can still explore"

**Phase:** MVP

---

### T3.4 — Vendor Menu Screen

**Description:** Full vendor detail with menu browsing. Header with cover image and vendor info, scrollable menu with category tabs, item list, and sticky cart button.

**Dependencies:** T2.3 (vendor/menu module), T2.9 (design system).

**Acceptance Criteria:**
- [ ] Collapsible header: cover image (200dp height), vendor name, cuisine, rating, delivery time, delivery fee
- [ ] Offer badges: e.g., "20% off (Min AED 50)" below vendor info
- [ ] Search menu bar: inline search filtering items by name
- [ ] Category tabs: horizontal scrollable, tap scrolls to category section
- [ ] Menu item row: name, description (truncated), price, "Add" button; tap opens item detail
- [ ] Item detail bottom sheet: full image, name, description, options with choices, quantity stepper, special instructions, "Add to Cart" button with total price
- [ ] Unavailable items shown greyed with "Unavailable" label (no Add button)
- [ ] Sticky bottom bar: "View Basket (2) · AED 85.00 →" with tap navigating to cart
- [ ] Add-to-cart animation: item count badge bounces on cart icon (optional, can be simple for MVP)

**Phase:** MVP

---

### T3.5 — Cart Screen

**Description:** Shows current cart contents with item management, voucher input, and price breakdown. Checkout CTA at bottom.

**Dependencies:** T2.4 (cart module), T2.9 (design system).

**Acceptance Criteria:**
- [ ] Vendor name at top of cart
- [ ] Cart item list: item name, selected options, quantity stepper, unit price, total price
- [ ] Swipe left to delete item (with undo snackbar)
- [ ] Voucher section: "Add voucher" expandable input with Apply button
- [ ] Price breakdown: Subtotal, Delivery Fee, Service Fee, Discount (if voucher applied), Total
- [ ] "Clear cart" option in top-right menu
- [ ] Empty cart state: illustration, "No items in cart", "Start ordering" CTA
- [ ] Checkout CTA: "Proceed to Checkout · AED {total}" at bottom
- [ ] Cross-vendor dialog: if user navigates to different vendor's menu, bottom sheet asks "Clear cart and start new order?"

**Phase:** MVP

---

### T3.6 — Checkout Screen

**Description:** Three-step checkout: address confirmation, payment method, and order placement. Streamlined for speed.

**Dependencies:** T2.5 (checkout module), T2.9 (design system).

**Acceptance Criteria:**
- [ ] Step indicator (3 dots or segments) at top
- [ ] Step 1 - Address: selected address card with "Change" link, address list if multiple
- [ ] Step 2 - Payment: saved cards list (radio selection), "Add new card" option, "Cash on delivery" option
- [ ] Step 3 - Confirm: order summary (items, pricing, address, payment method), "Place Order · AED {total}" CTA
- [ ] Payment processing: spinner overlay with "Processing payment..."
- [ ] Order success screen: checkmark animation, order number, "Track your order" primary CTA, "Back to home" secondary CTA
- [ ] Error handling: payment failed → red error card with retry/change payment options
- [ ] Add new card: card number input with brand detection, expiry, CVV, cardholder name (via Stripe/Checkout.com SDK)

**Phase:** MVP

---

### T3.7 — Order Tracking Screen

**Description:** Live order tracking with map, status timeline, and rider actions.

**Dependencies:** T2.6 (order tracking module), T2.9 (design system).

**Acceptance Criteria:**
- [ ] Map view: vendor pin, rider marker (motorcycle icon), delivery address pin
- [ ] Rider marker moves in real-time (Firebase RTDB listener)
- [ ] Status timeline: horizontal stepper with Ordered → Preparing → On the way → Delivered
- [ ] Current status highlighted with orange; completed steps with green checkmark
- [ ] Rider info card below map: name, vehicle type, rating, ETA
- [ ] Action buttons: Call (phone icon), Chat (message icon)
- [ ] Order details expandable section: vendor, items, pricing
- [ ] "Delivered" confirmation state with celebration animation
- [ ] Map supports zoom and pan gestures

**Phase:** MVP

---

### T3.8 — Order History Screen

**Description:** List of past and active orders with reorder functionality.

**Dependencies:** T2.7 (order history module), T2.9 (design system).

**Acceptance Criteria:**
- [ ] Filter tabs at top: All, Active, Completed, Cancelled
- [ ] Order card: vendor name + logo, item count, total, status badge, date/time
- [ ] Active orders show "Track" button
- [ ] Tap card → order detail screen with full breakdown
- [ ] "Reorder" button on completed orders: adds available items to cart
- [ ] Infinite scroll with loading indicator at bottom
- [ ] Empty state per filter tab

**Phase:** MVP

---

### T3.9 — Profile & Settings Screens

**Description:** User profile management, address book, payment methods, and settings.

**Dependencies:** T2.8 (profile module), T2.9 (design system).

**Acceptance Criteria:**
- [ ] Profile screen: avatar placeholder, name, email, phone, edit button
- [ ] Edit profile: first name, last name, email, DOB (date picker), gender, save/cancel
- [ ] Address book: list of addresses with edit/delete, add new address (map pin + form fields)
- [ ] Payment methods: list of saved cards (last 4, brand, expiry), add new card, delete with confirmation
- [ ] Settings: language toggle (English/Arabic), push notifications toggle, about, terms, privacy policy
- [ ] Logout button with confirmation dialog
- [ ] "Delete account" option with warning dialog (per app store requirements)

**Phase:** MVP

---

## MVP Components

### T3.10 — Core Component Library

**Description:** Build the essential reusable UI components needed by all screens. These form the foundation of the design system package.

**Dependencies:** None (foundational).

**Acceptance Criteria:**
- [ ] **Buttons**: Primary (orange bg, white text), Secondary (white bg, orange border), Tertiary (text only), Icon Button (circular)
- [ ] **Cards**: Vendor Card, Cart Item Card, Order Card, Address Card
- [ ] **Input Fields**: Phone Number (country flag + dial code), OTP (6 boxes), Text Input (with label, error, helper), Search Bar
- [ ] **Bottom Sheets**: Standard bottom sheet with drag handle, Item Detail sheet, Address Selector sheet, Confirmation sheet
- [ ] **Navigation**: Bottom Navigation Bar (4 tabs: Home, Search, Orders, Profile), Top App Bar (with back button, title, actions)
- [ ] **Status**: Badges (colored dots + text for order status), Tags (offer badges, "Coming soon"), Rating (star + number + count)
- [ ] **Feedback**: Loading spinner, Shimmer skeleton, Error card, Empty state (illustration + text + CTA), Toast/Snackbar
- [ ] **Stepper**: Quantity stepper (+/-), Checkout step indicator

**Phase:** MVP

---

## Phase 2 UI/UX Items (Not for MVP)

### T3.P2.1 — DineOut Screens
**Description:** Reservation booking flow, restaurant detail with amenities, booking management, bill payment.
**Phase:** Phase 2

### T3.P2.2 — Grocery/Q-Commerce Screens
**Description:** Product listing with filters, shopping list, item replacement timer, freshness guarantee, verified picker tracking.
**Phase:** Phase 2

### T3.P2.3 — Wallet & BNPL Screens
**Description:** Wallet dashboard, top-up flow, transaction history, BNPL dashboard, installment payment, Rewind feature.
**Phase:** Phase 2

### T3.P2.4 — Pro Subscription Screens
**Description:** Plan selection, subscription management, family plan, cancellation survey.
**Phase:** Phase 2

### T3.P2.5 — Rewards Screens
**Description:** Points balance, burn options, charity, raffle, voucher redemption.
**Phase:** Phase 2

### T3.P2.6 — AI Chat Interface
**Description:** ChatGPT-powered conversational food discovery with disclaimer and "Powered by ChatGPT" footer.
**Phase:** Phase 2

### T3.P2.7 — Advanced Interactions
**Description:** Micro-interactions (add-to-cart fly animation, item parallax), Lottie animations (order success, Pro activation), foldable device support.
**Phase:** Phase 2

### T3.P2.8 — Accessibility Enhancements
**Description:** Full screen reader support with accessibility labels, high contrast mode, font scaling, dedicated accessibility icon system.
**Phase:** Phase 2

### T3.P2.9 — Full Notification Center
**Description:** In-app notification list with Braze content cards, notification preferences per channel, smart suppression.
**Phase:** Phase 2

### T3.P2.10 — Advanced Search UI
**Description:** Autocomplete dropdown, multi-search shopping list, photo-to-list, filter bottom sheet, map view for vendors.
**Phase:** Phase 2
