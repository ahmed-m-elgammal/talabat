# 02 — Frontend MVP Scope

## 1. Overview

This plan defines the frontend implementation scope for the Talabat-like MVP. The mobile application is built with **Flutter** following a feature-first modular architecture with BLoC/Cubit state management, as described in `docs/12_frontend_architecture.md`. The MVP focuses on the customer-facing app with essential vendor portal and rider app screens.

The MVP app must support the complete order lifecycle with offline-first resilience, RTL (Arabic) layout support, and real-time order tracking. It targets both Android and iOS with a single codebase.

---

## 2. Feature Module Breakdown

### 2.1 MVP Feature Modules

| Module | Screens | Key BLoCs | Priority |
|--------|---------|-----------|----------|
| **auth** | Login, OTP, Registration, Social Login | AuthBloc, OtpBloc | P0 |
| **home** | Home Screen | HomeBloc, LocationBloc | P0 |
| **search** | Search, Results, Autocomplete | SearchBloc | P0 |
| **vendor** | Vendor Detail, Menu | VendorBloc, MenuBloc | P0 |
| **cart** | Cart, Cross-vendor dialog | CartBloc | P0 |
| **checkout** | Address Selection, Payment, Confirmation | CheckoutBloc, PaymentBloc | P0 |
| **order** | Order History, Order Detail, Tracking | OrderBloc, TrackingBloc | P0 |
| **payment** | Card Entry, Payment Methods | PaymentBloc | P0 |
| **profile** | Profile, Addresses, Settings | ProfileBloc, AddressBloc | P0 |
| **notifications** | Notification Center | NotificationBloc | P1 |
| **favorites** | Favorites List | FavoritesBloc | P1 |
| **chat** | Rider-Customer Chat | ChatBloc | P1 |

### 2.2 Phase 2 Modules (Excluded from MVP)

- wallet (talabat Pay)
- bnpl (PostPaid)
- subscription (Pro)
- rewards (Loyalty points)
- dineout (Reservations)
- pharmacy (Prescriptions)
- qcommerce (Full grocery with pickers)
- ai_chat (ChatGPT integration)

---

## 3. Project Structure (MVP)

```
talabat-mvp/
├── lib/
│   ├── main.dart
│   ├── app.dart
│   ├── core/
│   │   ├── network/
│   │   │   ├── api_client.dart          # Dio HTTP client with interceptors
│   │   │   ├── auth_interceptor.dart    # JWT token injection + refresh
│   │   │   └── error_handler.dart       # Unified error handling
│   │   ├── storage/
│   │   │   ├── local_storage.dart       # SharedPreferences abstraction
│   │   │   └── secure_storage.dart      # Token storage (flutter_secure_storage)
│   │   ├── analytics/
│   │   │   └── analytics_service.dart   # Event tracking (simplified Perseus)
│   │   ├── navigation/
│   │   │   └── app_router.dart          # GoRouter configuration
│   │   ├── theme/
│   │   │   ├── app_theme.dart           # Light/dark themes
│   │   │   ├── app_colors.dart          # Brand colors
│   │   │   └── app_typography.dart      # Font definitions
│   │   ├── utils/
│   │   │   ├── validators.dart          # Phone, email, OTP validation
│   │   │   ├── formatters.dart          # Currency, phone formatting
│   │   │   └── extensions.dart          # Common extensions
│   │   └── firebase/
│   │       ├── firebase_service.dart    # RTDB initialization
│   │       └── tracking_listener.dart   # Order tracking real-time
│   ├── features/
│   │   ├── auth/
│   │   │   ├── presentation/
│   │   │   │   ├── pages/              # Login, OTP, Registration
│   │   │   │   ├── widgets/            # OTP input, social buttons
│   │   │   │   └── bloc/
│   │   │   ├── domain/
│   │   │   │   ├── entities/
│   │   │   │   ├── repositories/
│   │   │   │   └── usecases/
│   │   │   ├── data/
│   │   │   │   ├── models/
│   │   │   │   ├── repositories/
│   │   │   │   └── datasources/
│   │   │   └── di/
│   │   ├── home/
│   │   ├── search/
│   │   ├── vendor/
│   │   ├── cart/
│   │   ├── checkout/
│   │   ├── order/
│   │   ├── payment/
│   │   ├── profile/
│   │   ├── notifications/
│   │   ├── favorites/
│   │   └── chat/
│   └── di/
│       └── injection.dart              # GetIt service locator
├── packages/
│   └── design_system/                  # Shared UI components
├── assets/
│   ├── i18n/
│   │   ├── en.json
│   │   └── ar.json
│   ├── images/
│   └── animations/
└── test/
    ├── unit/
    ├── widget/
    └── integration/
```

---

## 4. Tasks

### 4.1 Core Infrastructure Tasks

---

#### Task F-CORE-01: Project Scaffolding & DI Setup
**Description**: Initialize Flutter project with the feature-first modular structure. Configure GetIt for dependency injection, set up feature module folder structure, configure analysis_options.yaml with strict linting.
**Dependencies**: None
**Acceptance Criteria**:
- Flutter project created with directory structure matching spec
- GetIt service locator configured with all core services registered
- Analysis options configured with flutter_lints + custom rules
- Feature module template created (empty skeleton for each module)
- CI/CD pipeline configured (GitHub Actions) for build + test

---

#### Task F-CORE-02: API Client & Network Layer
**Description**: Implement Dio-based API client with JWT interceptor (auto-attach token, auto-refresh on 401), error handling interceptor (map HTTP errors to domain exceptions), logging interceptor, and connectivity check. Support offline mode with cached responses.
**Dependencies**: F-CORE-01
**Acceptance Criteria**:
- Dio client configured with base URL from environment config
- Auth interceptor: attaches Bearer token to all requests (except auth endpoints)
- Token refresh: on 401, calls refresh endpoint, retries original request
- Error mapping: HTTP errors → domain-specific exceptions with user-friendly messages
- Connectivity check: detect offline state, show banner
- Request timeout: 30s default, 60s for search
- Correlation ID header added to all requests

---

#### Task F-CORE-03: Theme & Design System Package
**Description**: Create the `design_system` package with brand colors, typography, spacing tokens, and reusable components (buttons, cards, input fields, bottom sheets) following `docs/13_ui_ux_specifications.md`. Support light/dark themes and RTL layout.
**Dependencies**: F-CORE-01
**Acceptance Criteria**:
- Brand colors defined: Orange (#FF5A00), Dark, Gray, Success, Warning, Error
- Typography scale: Display → Caption using TTCommonsPro (English) + NotoSansArabic (Arabic)
- Spacing tokens: xs(4dp) → 3xl(48dp)
- Corner radius tokens: none(0dp) → full(50%)
- Primary CTA, Secondary CTA, Tertiary, Icon, Floating CTA button components
- Vendor Card, Order Card, Menu Item Card components
- Input fields: Phone, OTP, Email, Password, Address Search, Card Number
- Bottom Sheet component with drag handle
- All components support RTL layout
- Light and dark themes configured

---

#### Task F-CORE-04: Navigation & Routing
**Description**: Configure GoRouter with all MVP routes, deep link support (`talabat://` scheme), and authentication guards. Routes must match the production navigation architecture from `docs/12_frontend_architecture.md` Section 6.
**Dependencies**: F-CORE-01
**Acceptance Criteria**:
- All routes defined: `/`, `/search`, `/vendor/{id}`, `/cart`, `/checkout`, `/orders`, `/orders/{id}`, `/orders/{id}/tracking`, `/profile`, `/auth/login`, `/auth/signup`
- Auth guard: unauthenticated users redirected to login for protected routes
- Deep links: `talabat://orders/{id}/tracking` opens order tracking
- URL-based navigation works on all platforms
- Navigation stack preserved correctly on deep link

---

#### Task F-CORE-05: Local Storage & Offline Foundation
**Description**: Implement local storage layer using SharedPreferences for simple key-values and SQLite (sqflite) for structured data. Set up cache tables for vendors, menus, and cart. Implement TTL-based cache invalidation.
**Dependencies**: F-CORE-01
**Acceptance Criteria**:
- SharedPreferences wrapper for: auth tokens, user preferences, feature flags, onboarding state
- SQLite database with tables: `cached_vendors`, `cached_menus`, `cart_items`, `search_history`, `order_history`
- Cache invalidation: TTL-based with configurable durations per data type
- Menu cache: 5-minute TTL with ETag validation
- Vendor cache: 5-minute TTL
- Cart persistence: survives app restarts
- Offline fallback: serve cached data when API unreachable with toast notification

---

#### Task F-CORE-06: Firebase RTDB Integration
**Description**: Configure Firebase SDK for Flutter. Set up RTDB listeners for order tracking (status + rider location). Handle offline persistence, automatic reconnection, and listener lifecycle management.
**Dependencies**: F-CORE-01
**Acceptance Criteria**:
- Firebase initialized in app startup
- Custom auth token obtained from backend for Firebase auth
- ValueEventListener on `order_tracking/{order_id}/status`
- ValueEventListener on `order_tracking/{order_id}/location`
- Offline disk persistence enabled
- Listeners attached when entering tracking screen, detached on exit
- Graceful handling of RTDB connection state changes

---

### 4.2 Auth Feature Tasks

---

#### Task F-AUTH-01: Login & OTP Screens
**Description**: Implement login screen with phone number input, OTP verification screen with 6-digit auto-fill boxes. Follow the production UI spec from `docs/13_ui_ux_specifications.md`. Support both login and registration flows.
**Dependencies**: F-CORE-02, F-CORE-03
**Acceptance Criteria**:
- Login screen: phone number input with country flag + dial code
- OTP screen: 6 individual digit boxes with auto-fill via sms_autofill
- Resend OTP button with countdown timer (60 seconds)
- OTP validation: auto-submit when all 6 digits entered
- Error states: invalid OTP, too many attempts, network error
- Success: store tokens, navigate to home
- Guest mode: "Skip" button on login screen
- All text supports English + Arabic (RTL)

---

#### Task F-AUTH-02: Social Login Integration
**Description**: Implement Google Sign-In and Apple Sign-In buttons on login screen. Handle account linking flow when social account matches existing user.
**Dependencies**: F-AUTH-01
**Acceptance Criteria**:
- Google Sign-In using google_sign_in package
- Apple Sign-In using sign_in_with_apple package
- Social tokens sent to backend for verification
- New user: auto-registered with provider info
- Existing user: logged in with JWT tokens
- Account linking dialog if email/phone matches different account

---

#### Task F-AUTH-03: Registration Screen
**Description**: Implement registration form with fields: first_name, last_name, email, phone, password, date_of_birth. Include terms/privacy consent checkboxes.
**Dependencies**: F-AUTH-01
**Acceptance Criteria**:
- All required fields validated before submission
- Email format validation
- Password: min 8 chars with complexity
- Date of birth picker (must be 13+)
- Terms & privacy consent required
- Marketing opt-in checkbox (optional)
- Success: auto-login and navigate to home

---

### 4.3 Home Feature Tasks

---

#### Task F-HOME-01: Home Screen Layout
**Description**: Implement the home screen following `docs/13_ui_ux_specifications.md` Section 4.2. Include: location selector, search bar, vertical tabs (Food), active order card, offers carousel, vendor listing.
**Dependencies**: F-CORE-03, F-CORE-04, F-VENDOR-01 (for vendor cards)
**Acceptance Criteria**:
- Location bar at top: "Delivering to {area}" with dropdown
- Search bar with placeholder: "Search for food or groceries"
- Vertical tab row: Food (only vertical for MVP)
- Active order card (if any): status, vendor name, ETA, Track/Help buttons
- Offers carousel: horizontal swipe of vendor cards with offers
- Popular near you: vertical list of vendor cards
- Bottom navigation: Home, Search, Orders, Profile
- Pull-to-refresh refreshes all sections
- Skeleton/shimmer loading states for all sections

---

#### Task F-HOME-02: Location & Address Selection
**Description**: Implement address selection bottom sheet with saved addresses list and map-based address picker. Default address pre-selected.
**Dependencies**: F-HOME-01
**Acceptance Criteria**:
- Tapping location bar opens address bottom sheet
- Saved addresses listed with label, area, building info
- "Add new address" option opens map picker
- Map picker: drag pin to set location, reverse geocode to area name
- New address form: label, building, floor, apartment, delivery instructions
- Default address auto-selected on app launch
- Address selection triggers vendor list refresh for new area

---

### 4.4 Search Feature Tasks

---

#### Task F-SEARCH-01: Search Screen & Autocomplete
**Description**: Implement search screen with autocomplete suggestions, recent searches, and search results. Follow `docs/16_search_discovery_system.md` spec for autocomplete UI.
**Dependencies**: F-CORE-02, F-CORE-03
**Acceptance Criteria**:
- Search bar with auto-focus on screen open
- As user types: autocomplete suggestions appear (debounced 300ms)
- Recent searches shown (stored locally, max 10)
- Suggestions: vendor names, item names, category suggestions
- "Search for {query} in Food" deep link option
- Clear search button
- Empty state: "No results for {query}" with suggestions
- Search results: vendor cards + item cards (combined view)

---

#### Task F-SEARCH-02: Search Results & Filtering
**Description**: Implement search results view with vendor and item sections, sort/filter bottom sheet, and result caching.
**Dependencies**: F-SEARCH-01
**Acceptance Criteria**:
- Results split into: Restaurants section + Dishes section
- Sort options: Recommended, Delivery time, Rating, Distance
- Filter options: Cuisine, Offers, Free delivery, Open now
- Active filters shown as chips with remove option
- Results cached locally for 5 minutes
- Cached results shown when offline with appropriate messaging
- Tap vendor card → navigate to vendor detail
- Tap item card → navigate to vendor menu (scroll to item)

---

### 4.5 Vendor & Menu Feature Tasks

---

#### Task F-VENDOR-01: Vendor Detail & Menu Screen
**Description**: Implement vendor detail screen with cover image, info section, and scrollable menu following `docs/13_ui_ux_specifications.md` Section 4.3. Menu items shown in categories with Add buttons.
**Dependencies**: F-CORE-02, F-CORE-03
**Acceptance Criteria**:
- Cover image (200dp) with back button overlay
- Vendor info: name, cuisine, rating, delivery time, delivery fee
- Offer badges (if any)
- Menu search bar
- Category headers as sticky section dividers
- Menu items: name, description, price, Add button
- Unavailable items: shown greyed with "Out of stock" label
- Floating cart button at bottom: "View Basket ({count}) - {total}"
- Pull-to-refresh refreshes menu data
- Skeleton loading for initial menu load

---

#### Task F-VENDOR-02: Item Detail Bottom Sheet
**Description**: Implement item detail bottom sheet with image, description, option selection (required/optional), quantity stepper, special instructions, and Add to Cart button. Follow the production spec from `docs/13_ui_ux_specifications.md`.
**Dependencies**: F-VENDOR-01, F-CART-01
**Acceptance Criteria**:
- Bottom sheet opens on item tap
- Item image, name, description, base price displayed
- Required options: must select before Add button enabled
- Optional options: can skip or select up to max_selections
- Price updates dynamically based on option choices
- Quantity stepper: +/- buttons with min 1
- Special instructions text field (max 200 chars)
- "Add to Cart - {total_price}" button at bottom
- Cart badge updates with bounce animation on add
- Accessibility: "Item successfully added to cart" announcement

---

### 4.6 Cart Feature Tasks

---

#### Task F-CART-01: Cart Screen
**Description**: Implement cart screen showing items from current vendor, with quantity controls, item removal (swipe-to-delete), subtotal, delivery fee, service fee, voucher field, and checkout CTA.
**Dependencies**: F-CORE-02, F-CORE-03
**Acceptance Criteria**:
- Items listed with name, selected options, quantity, price
- Quantity stepper (+/-) on each item
- Swipe-to-delete with confirmation
- Subtotal, delivery fee, service fee, discount (if voucher), total
- Voucher input field with "Apply" button
- Checkout CTA: "Place Order - {total}"
- Cross-vendor dialog: "Clear cart and start new order?" when switching vendors
- Empty cart state: "Your cart is empty"
- Cart persists across app restarts (server-side + local)

---

#### Task F-CART-02: Voucher Application
**Description**: Implement voucher code input with validation. Show discount applied or error message. Support BIN-voucher error handling (payment method mismatch).
**Dependencies**: F-CART-01
**Acceptance Criteria**:
- Voucher input field: uppercase transformation
- "Apply" button triggers server validation
- Success: discount shown in price breakdown
- Error: "Invalid voucher code" / "Voucher expired" / "Minimum order not met"
- BIN voucher error: "This voucher requires a specific card type" with change/remove options
- Only one voucher per cart
- Voucher removal: tap X on applied voucher

---

### 4.7 Checkout Feature Tasks

---

#### Task F-CHECKOUT-01: Checkout Flow
**Description**: Implement multi-step checkout: Address confirmation → Time selection (ASAP only for MVP) → Payment selection → Order confirmation. Follow production spec from `docs/06_order_management_system.md` Section 4.
**Dependencies**: F-CART-01, F-PAY-01
**Acceptance Criteria**:
- Step 1: Address pre-selected (default), change address button
- Step 2: Delivery time "ASAP" (estimated {min}-{max} min)
- Step 3: Payment method selection (card, cash, add new card)
- Order summary: items, pricing breakdown
- "Place Order" CTA with price
- Loading state during order placement
- Success: navigate to order tracking
- Error: specific messages for payment failure, item unavailable, etc.
- Offline: "You're offline. Check your connection." message

---

### 4.8 Payment Feature Tasks

---

#### Task F-PAY-01: Payment Methods Screen
**Description**: Implement payment method selection screen. Show saved cards (masked), Apple Pay / Google Pay, cash, and "Add new card" option. Follow production payment flow from `docs/14_payment_system.md`.
**Dependencies**: F-CORE-02, F-CORE-03
**Acceptance Criteria**:
- Saved cards shown with last four digits, brand icon, expiry
- Apple Pay button (iOS only, conditional availability check)
- Google Pay button (Android only, conditional availability check)
- Cash on Delivery option
- "Add new card" opens card entry form
- Default payment method pre-selected
- Card entry uses Stripe.js tokenization (no raw data to our server)

---

#### Task F-PAY-02: Card Entry & 3DS
**Description**: Implement card entry form with Stripe card element. Handle 3D Secure authentication flow for card payments.
**Dependencies**: F-PAY-01
**Acceptance Criteria**:
- Card number input with brand detection and formatting
- Expiry date input (MM/YY)
- CVV input (obscured)
- Cardholder name input
- Real-time validation: Luhn check, expiry check
- 3DS challenge: open in WebView if required by issuer
- 3DS success: payment proceeds
- 3DS failure: "Authentication failed, try again"
- Token stored for future use (save card checkbox)

---

### 4.9 Order Feature Tasks

---

#### Task F-ORDER-01: Order History Screen
**Description**: Implement order history list with order cards showing vendor name, items summary, total, status, date. Support reorder action.
**Dependencies**: F-CORE-02, F-CORE-03
**Acceptance Criteria**:
- Orders listed chronologically (most recent first)
- Each card: vendor name, items count, total, status badge, date
- Active orders shown at top with "Track" button
- Past orders shown with "Reorder" button
- Pull-to-refresh
- Empty state: "No orders yet" with "Start ordering" CTA
- Pagination: load more on scroll

---

#### Task F-ORDER-02: Order Detail Screen
**Description**: Implement order detail screen showing full order info: status timeline, items, pricing, delivery info, rider info (if assigned), and action buttons (track, cancel, reorder).
**Dependencies**: F-ORDER-01
**Acceptance Criteria**:
- Order status timeline with visual indicators
- Items list with quantities, options, prices
- Pricing breakdown: subtotal, delivery fee, service fee, discount, tip, total
- Delivery address
- Rider info: name, vehicle, rating (when assigned)
- Cancel button (visible only during cancellation window)
- Reorder button
- Help / support link
- Order modification display (if any modifications occurred)

---

#### Task F-ORDER-03: Order Tracking Screen with Live Map
**Description**: Implement real-time order tracking screen following `docs/13_ui_ux_specifications.md` Section 4.4. Include live map with rider position, status indicator, ETA, and action buttons (call, chat, tip).
**Dependencies**: F-ORDER-02, F-CORE-06
**Acceptance Criteria**:
- Map view showing: vendor location, rider position, delivery address
- Rider icon moves in real-time (updated via Firebase RTDB listener)
- Status indicator: Ordered / Preparing / On the way / Delivered
- Rider info: name, vehicle, rating
- ETA display: "12 minutes" (updates every 30 seconds)
- Action buttons: Call rider, Chat, Tip (after delivery)
- Order details expandable section below map
- Map uses Google Maps (Google Play Services devices)
- Offline: last known rider position shown with "Reconnecting..." indicator

---

### 4.10 Profile Feature Tasks

---

#### Task F-PROFILE-01: Profile & Settings Screen
**Description**: Implement profile screen showing user info, saved addresses, payment methods, notification preferences, and app settings.
**Dependencies**: F-CORE-02, F-CORE-03
**Acceptance Criteria**:
- User info: name, email, phone (editable)
- Saved addresses list with edit/delete
- Payment methods list with add/remove
- Notification preferences toggle
- Language selection (English / Arabic)
- Dark mode toggle
- Logout button
- About / Legal links

---

### 4.11 Chat Feature Tasks (P1)

---

#### Task F-CHAT-01: Rider-Customer Chat
**Description**: Implement in-app chat between customer and rider during active delivery. Use Firebase RTDB for real-time messaging. Support predefined quick messages and text input.
**Dependencies**: F-CORE-06, F-ORDER-03
**Acceptance Criteria**:
- Chat screen accessible from order tracking screen
- Real-time messages via Firebase RTDB listener
- Predefined messages: "I'm at the gate", "Come to lobby", "Leave at door"
- Text input with send button
- Message timestamps and read status
- Push notification for new messages when chat screen not visible
- Chat disabled after order delivered (read-only)

---

## 5. State Management Patterns

### 5.1 BLoC/Cubit Convention

Every feature module uses BLoC/Cubit following this pattern:

```
FeatureBloc / FeatureCubit
├── Events/States clearly typed
├── Delegates to use cases for business logic
├── Emits new states on event processing
├── Never directly accesses repositories (use cases mediate)
```

### 5.2 Key BLoC Interactions

```
CartBloc ──── adds item ────→ MenuBloc (update item count in menu)
CartBloc ──── clears cart ────→ VendorBloc (reset vendor context)
OrderBloc ──── places order ───→ CartBloc (clear cart on success)
TrackingBloc ──── status change ───→ HomeBloc (update active order card)
PaymentBloc ──── payment success ───→ OrderBloc (confirm order)
```

---

## 6. Performance Targets

| Metric | Target (High-End) | Target (Low-End) | Measurement |
|--------|--------------------|-------------------|-------------|
| App startup TTI | < 1.5s | < 3s | Time to interactive home screen |
| Home screen TTI | < 1s | < 2s | Time from navigation to interactive |
| Menu load time | < 1s | < 2s | Time from tap to menu rendered |
| Order placement | < 2s | < 3s | Time from tap to confirmation |
| Search results | < 500ms | < 1s | Time from query to results |
| Tracking map update | < 2s | < 3s | Rider position update latency |

---

## 7. Testing Strategy

| Type | Framework | Coverage Target | Scope |
|------|-----------|----------------|-------|
| Unit tests | flutter_test | 80%+ | BLoCs, use cases, repositories, mappers |
| Widget tests | flutter_test | 70%+ | Individual widgets, screens |
| Integration tests | integration_test | Key flows | Login → Order → Track |
| Golden tests | golden_toolkit | Core components | Design system components |

### Critical Integration Test Flows

1. **Happy path**: Login → Browse → Add to cart → Checkout → Track order → Delivery
2. **Offline resilience**: Browse → Lose network → Cart operations → Reconnect → Sync
3. **RTL layout**: Switch to Arabic → All screens display correctly
4. **Payment flow**: Add card → 3DS challenge → Payment success/failure
