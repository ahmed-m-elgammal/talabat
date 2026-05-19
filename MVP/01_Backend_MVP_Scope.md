# 01 — Backend MVP Scope

## 1. Overview

This plan defines the backend implementation scope for the Talabat-like MVP. The backend follows a **modular monolith** architecture deployed as a single service with clearly separated domain modules. Each module owns its database tables, business logic, and API surface. Inter-module communication uses in-process function calls (synchronous) and an internal event bus (asynchronous) that can later be replaced with Kafka when extracting to microservices.

The MVP backend must support the complete order lifecycle for 50-100 food vendors with real-time inventory, ordering, payment processing, dispatch, and tracking. It must be ready for horizontal scaling via load-balanced instances sharing a PostgreSQL database.

---

## 2. Backend Module Map

| Module | Bounded Context | Primary Tables | Key Responsibilities |
|--------|----------------|---------------|---------------------|
| **Auth Module** | User identity & sessions | `users`, `user_auth_providers`, `sessions` | OTP send/verify, JWT issuance, token refresh |
| **User Module** | User profiles & addresses | `profiles`, `addresses` | Profile CRUD, address management, favorites |
| **Vendor Module** | Restaurant/store management | `vendors`, `vendor_schedules`, `menu_categories`, `menu_items`, `item_options`, `option_choices` | Vendor CRUD, menu management, operating hours |
| **Inventory Module** | Stock & availability | `stock_reservations`, `inventory_audit_log` | Stock check, reservation, deduction, release |
| **Cart Module** | Shopping cart | `carts`, `cart_items` | Add/remove/update items, cross-vendor handling, server-side cart |
| **Order Module** | Order lifecycle | `orders`, `order_items`, `order_modifications`, `order_timeline` | Order creation, status transitions, cancellation, reorder |
| **Payment Module** | Payment processing | `payments`, `payment_tokens` | Stripe integration, cash handling, refund processing |
| **Dispatch Module** | Rider assignment & tracking | `riders`, `delivery_assignments` | Rider matching, assignment, ETA calculation |
| **Search Module** | Discovery | (Reads from vendor tables) | Full-text search, filtering, sorting, autocomplete |
| **Notification Module** | Push & in-app messages | `notification_log`, `device_tokens` | FCM push, order status alerts |
| **Voucher Module** | Promotions | `vouchers`, `voucher_usage` | Voucher validation, BIN restriction, usage tracking |

---

## 3. API Design

### 3.1 API Versioning & Conventions

Follow the production API conventions from `docs/10_api_specification.md`:

- Base URL: `https://api.{domain}/v1`
- JSON request/response payloads
- JWT Bearer authentication
- Standard HTTP status codes
- Correlation IDs (`X-Correlation-ID` header)
- ETag-based caching for GET endpoints
- Rate limiting headers in responses

### 3.2 MVP Endpoint Summary

| Category | Endpoints | Auth Required |
|----------|-----------|--------------|
| Auth | `POST /auth/otp/send`, `POST /auth/otp/verify`, `POST /auth/token/refresh`, `POST /auth/social/login` | No (except refresh) |
| Users | `GET /users/me`, `PATCH /users/me`, `GET /users/me/addresses`, `POST /users/me/addresses`, `PUT /users/me/addresses/{id}`, `DELETE /users/me/addresses/{id}` | Yes |
| Vendors | `GET /vendors`, `GET /vendors/{id}`, `GET /vendors/{id}/menu` | No (public) |
| Search | `GET /search`, `GET /search/autocomplete` | No (public) |
| Cart | `GET /cart`, `POST /cart/items`, `PUT /cart/items/{id}`, `DELETE /cart/items/{id}`, `DELETE /cart`, `POST /cart/voucher` | Yes |
| Orders | `POST /orders`, `GET /orders`, `GET /orders/{id}`, `POST /orders/{id}/cancel`, `POST /orders/{id}/reorder` | Yes |
| Payments | `GET /payments/methods`, `POST /payments/tokenize/card`, `POST /payments/checkout`, `POST /payments/webhook/stripe` | Yes (except webhook) |
| Dispatch | `GET /dispatch/orders/{id}/tracking`, `POST /dispatch/riders/{id}/location`, `POST /dispatch/riders/{id}/status` | Yes (rider auth) |
| Notifications | `POST /notifications/device-token`, `GET /notifications/preferences`, `PUT /notifications/preferences` | Yes |
| Vendor Portal | `GET /portal/vendors/me`, `PATCH /portal/vendors/me`, `GET /portal/vendors/me/orders`, `PATCH /portal/vendors/me/orders/{id}/status`, `PATCH /portal/vendors/me/menu-items/{id}/availability` | Yes (vendor auth) |
| Rider App | `GET /rider/orders`, `PATCH /rider/orders/{id}/status`, `POST /rider/location`, `GET /rider/earnings` | Yes (rider auth) |

---

## 4. Database Schema (MVP)

The schema follows the production design from `docs/04_database_architecture.md`, simplified for single-country deployment:

### 4.1 Core Tables

```sql
-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number VARCHAR(20) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255),
    country_code CHAR(2) DEFAULT 'AE',
    date_of_birth DATE,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    label VARCHAR(20) NOT NULL,
    area_name VARCHAR(100) NOT NULL,
    latitude DECIMAL(10,7) NOT NULL,
    longitude DECIMAL(10,7) NOT NULL,
    building VARCHAR(100),
    floor VARCHAR(20),
    apartment VARCHAR(20),
    delivery_instructions TEXT,
    is_default BOOLEAN DEFAULT false,
    geohash VARCHAR(12),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Vendors
CREATE TABLE vendors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID REFERENCES users(id),
    name VARCHAR(200) NOT NULL,
    name_ar VARCHAR(200),
    vertical_type VARCHAR(20) DEFAULT 'food',
    latitude DECIMAL(10,7) NOT NULL,
    longitude DECIMAL(10,7) NOT NULL,
    delivery_radius_km DECIMAL(5,2) DEFAULT 5.0,
    is_open BOOLEAN DEFAULT true,
    is_busy BOOLEAN DEFAULT false,
    delivery_fee_base DECIMAL(10,2) DEFAULT 5.00,
    minimum_order_value DECIMAL(10,2) DEFAULT 0.00,
    estimated_delivery_time_min INT DEFAULT 30,
    estimated_delivery_time_max INT DEFAULT 45,
    rating_avg DECIMAL(3,2) DEFAULT 0.0,
    rating_count INT DEFAULT 0,
    cuisine_types JSONB DEFAULT '[]',
    logo_url TEXT,
    cover_url TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE vendor_schedules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vendor_id UUID NOT NULL REFERENCES vendors(id) ON DELETE CASCADE,
    day_of_week INT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),
    open_time TIME NOT NULL,
    close_time TIME NOT NULL,
    is_closed BOOLEAN DEFAULT false
);

CREATE TABLE menu_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vendor_id UUID NOT NULL REFERENCES vendors(id) ON DELETE CASCADE,
    name VARCHAR(200) NOT NULL,
    name_ar VARCHAR(200),
    display_order INT DEFAULT 0,
    is_available BOOLEAN DEFAULT true
);

CREATE TABLE menu_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_id UUID NOT NULL REFERENCES menu_categories(id) ON DELETE CASCADE,
    vendor_id UUID NOT NULL REFERENCES vendors(id) ON DELETE CASCADE,
    name VARCHAR(200) NOT NULL,
    name_ar VARCHAR(200),
    description TEXT,
    description_ar TEXT,
    base_price DECIMAL(10,2) NOT NULL,
    is_available BOOLEAN DEFAULT true,
    stock_count INT,
    image_url TEXT,
    preparation_time_minutes INT DEFAULT 10,
    display_order INT DEFAULT 0
);

CREATE TABLE item_options (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id UUID NOT NULL REFERENCES menu_items(id) ON DELETE CASCADE,
    name VARCHAR(200) NOT NULL,
    name_ar VARCHAR(200),
    is_required BOOLEAN DEFAULT false,
    min_selections INT DEFAULT 0,
    max_selections INT DEFAULT 1,
    display_order INT DEFAULT 0
);

CREATE TABLE option_choices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    option_id UUID NOT NULL REFERENCES item_options(id) ON DELETE CASCADE,
    name VARCHAR(200) NOT NULL,
    name_ar VARCHAR(200),
    price_modifier DECIMAL(10,2) DEFAULT 0.00,
    is_default BOOLEAN DEFAULT false,
    is_available BOOLEAN DEFAULT true
);

-- Orders
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number VARCHAR(30) UNIQUE NOT NULL,
    user_id UUID NOT NULL REFERENCES users(id),
    vendor_id UUID NOT NULL REFERENCES vendors(id),
    status VARCHAR(20) NOT NULL DEFAULT 'ordered',
    delivery_type VARCHAR(10) DEFAULT 'delivery',
    subtotal DECIMAL(10,2) NOT NULL,
    delivery_fee DECIMAL(10,2) DEFAULT 0.00,
    service_fee DECIMAL(10,2) DEFAULT 0.00,
    discount_amount DECIMAL(10,2) DEFAULT 0.00,
    voucher_code VARCHAR(50),
    rider_tip DECIMAL(10,2) DEFAULT 0.00,
    total DECIMAL(10,2) NOT NULL,
    payment_method VARCHAR(20) NOT NULL,
    payment_status VARCHAR(20) DEFAULT 'pending',
    delivery_address_snapshot JSONB NOT NULL,
    delivery_latitude DECIMAL(10,7),
    delivery_longitude DECIMAL(10,7),
    cancellation_reason TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    menu_item_id UUID REFERENCES menu_items(id),
    item_name VARCHAR(200) NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) NOT NULL,
    selected_options JSONB DEFAULT '[]',
    special_instructions TEXT
);

-- Payments
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    user_id UUID NOT NULL REFERENCES users(id),
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'AED',
    payment_method_type VARCHAR(20) NOT NULL,
    gateway VARCHAR(20) NOT NULL,
    gateway_transaction_id VARCHAR(255),
    status VARCHAR(20) NOT NULL DEFAULT 'initiated',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE payment_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    token_type VARCHAR(20) NOT NULL,
    token_value TEXT NOT NULL,
    card_last_four VARCHAR(4),
    card_brand VARCHAR(20),
    is_default BOOLEAN DEFAULT false,
    provider VARCHAR(20) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Dispatch
CREATE TABLE riders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone_number VARCHAR(20) NOT NULL,
    vehicle_type VARCHAR(20) DEFAULT 'motorcycle',
    is_active BOOLEAN DEFAULT true,
    current_latitude DECIMAL(10,7),
    current_longitude DECIMAL(10,7),
    status VARCHAR(20) DEFAULT 'offline',
    rating_avg DECIMAL(3,2) DEFAULT 5.0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE delivery_assignments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    rider_id UUID NOT NULL REFERENCES riders(id),
    status VARCHAR(20) NOT NULL DEFAULT 'assigned',
    assigned_at TIMESTAMPTZ DEFAULT NOW(),
    picked_up_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ
);

-- Vouchers
CREATE TABLE vouchers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(50) UNIQUE NOT NULL,
    discount_type VARCHAR(20) NOT NULL,
    discount_value DECIMAL(10,2) NOT NULL,
    min_order_value DECIMAL(10,2) DEFAULT 0.00,
    max_discount_cap DECIMAL(10,2),
    usage_limit_total INT,
    usage_limit_per_user INT DEFAULT 1,
    usage_count INT DEFAULT 0,
    valid_from TIMESTAMPTZ NOT NULL,
    valid_until TIMESTAMPTZ NOT NULL,
    is_active BOOLEAN DEFAULT true
);

-- Notifications
CREATE TABLE notification_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    type VARCHAR(50) NOT NULL,
    channel VARCHAR(20) NOT NULL,
    title VARCHAR(200),
    body TEXT,
    data JSONB,
    sent_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 4.2 Indexes

```sql
CREATE INDEX idx_vendors_geo ON vendors USING gist(ll_to_earth(latitude::float8, longitude::float8));
CREATE INDEX idx_vendors_open ON vendors(is_open, vertical_type) WHERE is_open = true;
CREATE INDEX idx_orders_user ON orders(user_id, created_at DESC);
CREATE INDEX idx_orders_vendor ON orders(vendor_id, status);
CREATE INDEX idx_menu_items_vendor ON menu_items(vendor_id, is_available);
CREATE INDEX idx_delivery_assignments_rider ON delivery_assignments(rider_id, status);
```

---

## 5. Tasks

### 5.1 Auth Module Tasks

---

#### Task B-AUTH-01: OTP Send/Verify API
**Description**: Implement `POST /auth/otp/send` and `POST /auth/otp/verify` endpoints. Generate 6-digit OTP with 5-minute expiry, store in Redis with key `otp:{phone}:{request_id}`, rate-limit to 3 requests per phone per 5 minutes. On verify, issue JWT access token (1h) and refresh token (30d) using RS256. Store refresh token hash in `sessions` table.
**Dependencies**: None
**Acceptance Criteria**:
- OTP is sent via SMS gateway (Twilio/Vonage for MVP)
- OTP stored in Redis with 5-minute TTL
- Rate limiting enforced: 4th request within 5 min returns 429
- Valid OTP returns access_token + refresh_token in response
- Invalid OTP (after 5 attempts) invalidates the request
- JWT tokens follow the structure from `docs/08_authentication_authorization.md`
- Refresh token stored with device_id for multi-device support

---

#### Task B-AUTH-02: Social Login (Google + Apple)
**Description**: Implement `POST /auth/social/login` for Google and Apple ID tokens. Verify the ID token with the provider's public keys, create or link user account, issue JWT tokens. Follow the production flow from `docs/08_authentication_authorization.md` Section 2.3.
**Dependencies**: B-AUTH-01
**Acceptance Criteria**:
- Google ID token verified against Google's public keys
- Apple ID token verified against Apple's JWKS
- New user automatically created with provider info
- Existing user linked if phone/email matches
- JWT tokens issued on success

---

#### Task B-AUTH-03: Token Refresh & Session Management
**Description**: Implement `POST /auth/token/refresh`. Validate refresh token, check session existence, issue new access + refresh tokens, rotate refresh token (old one invalidated). Implement session cleanup for expired sessions.
**Dependencies**: B-AUTH-01
**Acceptance Criteria**:
- Valid refresh token returns new access + refresh tokens
- Old refresh token is invalidated (single-use rotation)
- Expired refresh token returns 401 with clear error
- All in-flight requests retried with new token on client side
- Sessions cleaned up after 30 days of inactivity

---

#### Task B-AUTH-04: Guest Mode Support
**Description**: Allow unauthenticated access to public endpoints (vendor listing, menu, search). Rate-limit guest requests by IP (100 req/min). When guest attempts an authenticated action, return 401 with a flag indicating auth is required.
**Dependencies**: B-AUTH-01
**Acceptance Criteria**:
- Public endpoints accessible without Authorization header
- Guest users cannot access /cart, /orders, /payments, /addresses
- Guest rate limiting enforced at API gateway level
- Client can prompt login when 401 received on auth-required action

---

### 5.2 User Module Tasks

---

#### Task B-USER-01: User Profile CRUD
**Description**: Implement `GET /users/me`, `PATCH /users/me` for profile management. Support fields: first_name, last_name, email, date_of_birth, gender. Validate email uniqueness, date_of_birth (13+ years old).
**Dependencies**: B-AUTH-01
**Acceptance Criteria**:
- GET returns full profile (excluding password_hash)
- PATCH updates only provided fields (partial update)
- Email uniqueness validated on change
- Date of birth validated: user must be 13+
- Updated_at timestamp auto-updated

---

#### Task B-USER-02: Address Management
**Description**: Implement `GET/POST/PUT/DELETE /users/me/addresses`. Support geohash-based area identification. Enforce maximum 10 addresses per user. Default address logic: only one default at a time.
**Dependencies**: B-AUTH-01
**Acceptance Criteria**:
- Create address with lat/lng, area_name, building, floor, apartment
- Geohash computed from coordinates on insert
- Setting is_default=true clears previous default
- Maximum 10 addresses enforced (11th returns 422)
- Delete address recalculates default if deleted address was default

---

#### Task B-USER-03: Favorites System
**Description**: Implement `POST /users/me/favorites/{vendor_id}` and `GET /users/me/favorites`. Simple vendor favorite toggle for MVP.
**Dependencies**: B-USER-01, B-VENDOR-01
**Acceptance Criteria**:
- Add vendor to favorites (idempotent)
- Remove vendor from favorites
- List favorited vendors with basic info (name, logo, rating, is_open)
- Favorited vendors shown first in vendor listing (with feature flag)

---

### 5.3 Vendor Module Tasks

---

#### Task B-VENDOR-01: Vendor Listing API
**Description**: Implement `GET /vendors` with geo-based filtering (within delivery radius of user's address), pagination (default 20, max 50), and sorting (recommended, delivery_time, rating, distance). Use PostGIS for geospatial queries.
**Dependencies**: B-USER-02
**Acceptance Criteria**:
- Returns vendors within delivery_radius_km of user's coordinates
- Supports pagination with `page` and `limit` params
- Supports sort: recommended (composite score), delivery_time, rating, distance
- Filters: is_open, cuisine_types, offer_available
- Vendor card includes: name, name_ar, logo, rating, delivery_time, delivery_fee, is_open, is_busy, offers
- Response includes ETag for caching
- Cache vendor listing in Redis with 5-minute TTL keyed by area geohash

---

#### Task B-VENDOR-02: Vendor Menu API
**Description**: Implement `GET /vendors/{id}/menu`. Return full menu with categories, items, options, and choices. Support ETag-based conditional GET. Cache menu in Redis with content hash.
**Dependencies**: B-VENDOR-01
**Acceptance Criteria**:
- Returns vendor info + categories + items + options + choices
- Items include availability status (is_available + stock_count)
- Supports If-None-Match header for 304 Not Modified
- ETag computed from menu content hash
- Menu cached in Redis with 5-minute TTL
- Unavailable items still returned but flagged (frontend decides display)

---

#### Task B-VENDOR-03: Vendor Portal - Menu Management
**Description**: Implement vendor portal APIs for menu CRUD: create/update/delete categories, items, options, choices. Support bulk item availability toggle. Auth requires vendor_admin role.
**Dependencies**: B-VENDOR-02
**Acceptance Criteria**:
- Vendor admin can create/edit/delete menu categories
- Vendor admin can create/edit/delete menu items with options
- Item availability toggle (is_available flag)
- Stock count update for finite-stock items
- Changes invalidate menu cache in Redis
- Changes trigger `menu.updated` event for search index refresh

---

#### Task B-VENDOR-04: Vendor Portal - Order Management
**Description**: Implement `GET /portal/vendors/me/orders` and `PATCH /portal/vendors/me/orders/{id}/status`. Vendors can view incoming orders, accept (status → preparing), reject (status → cancelled), and mark ready for pickup.
**Dependencies**: B-VENDOR-01, B-ORDER-01
**Acceptance Criteria**:
- List orders filtered by status (pending, preparing, ready)
- Accept order changes status from 'ordered' to 'preparing'
- Reject order requires cancellation_reason
- Status change triggers notification to customer
- Status change publishes `order.status_changed` event
- Vendor can only see their own orders

---

### 5.4 Inventory Module Tasks

---

#### Task B-INV-01: Stock Reservation & Deduction
**Description**: Implement two-phase inventory model. On cart add: reserve stock with 15-minute TTL in Redis (`reservation:{item_id}:{cart_id}`). On order confirm: deduct from PostgreSQL stock_count atomically. On cart expire/cancel: release reservation.
**Dependencies**: B-VENDOR-02, B-CART-01
**Acceptance Criteria**:
- Cart add checks Redis for reservation, falls back to PostgreSQL
- For finite-stock items: DECREMENT stock_count atomically with optimistic locking
- For infinite-stock items (food): only check is_available flag
- Reservation created in Redis with 15-minute TTL
- Order confirm converts reservation to permanent deduction
- Cart abandon releases reservation after TTL expiry
- Out-of-stock during cart add returns 409 with unavailable item list

---

#### Task B-INV-02: Real-Time Availability Updates
**Description**: When vendor toggles item availability or stock reaches zero, update Redis cache and publish `inventory.updated` event. Customer app receives updates via SSE or next API poll refresh.
**Dependencies**: B-VENDOR-03, B-INV-01
**Acceptance Criteria**:
- Vendor availability change updates Redis immediately
- `inventory.updated` event published to internal event bus
- Event includes: vendor_id, item_id, is_available, stock_count
- Search index refreshed on availability change
- Cart items checked for availability on checkout initiation

---

### 5.5 Cart Module Tasks

---

#### Task B-CART-01: Cart CRUD Operations
**Description**: Implement `GET /cart`, `POST /cart/items`, `PUT /cart/items/{id}`, `DELETE /cart/items/{id}`, `DELETE /cart`. Single-vendor cart model: adding items from a different vendor triggers cart clear. Server-side cart stored in Redis with 24-hour TTL.
**Dependencies**: B-AUTH-01, B-VENDOR-02
**Acceptance Criteria**:
- Cart persists in Redis as `cart:{user_id}` with 24h TTL
- Add item validates: vendor exists, item exists, item is_available, options valid
- Cross-vendor add returns conflict with option to clear cart
- Update quantity recalculates subtotal
- Remove item recalculates subtotal
- Clear cart removes all items and vendor association
- Cart response includes subtotal, delivery_fee, service_fee, total

---

#### Task B-CART-02: Voucher Application
**Description**: Implement `POST /cart/voucher`. Validate voucher code, check expiry, usage limits, min_order_value. Apply discount to cart total. Support percentage and fixed_amount discount types.
**Dependencies**: B-CART-01
**Acceptance Criteria**:
- Valid voucher applies discount to cart
- Voucher not found returns 404
- Expired voucher returns 422 with message
- Usage limit exceeded returns 422
- Min order value not met returns 422 with required amount
- Only one voucher per cart
- Discount reflected in cart total calculation

---

### 5.6 Order Module Tasks

---

#### Task B-ORDER-01: Order Creation
**Description**: Implement `POST /orders`. Validate cart, check item availability, calculate pricing (subtotal + delivery_fee + service_fee - discount + tip = total), reserve stock, initiate payment, create order record, publish `order.created` event.
**Dependencies**: B-CART-01, B-INV-01, B-PAY-01
**Acceptance Criteria**:
- Order created with unique order_number (format: TB-{DATE}-{SEQ})
- All cart items verified available before order creation
- Stock reserved (not yet deducted) at order creation
- Payment initiated (auth only, capture on delivery for food)
- Order status set to 'ordered'
- `order.created` event published to internal bus
- Firebase RTDB path created for tracking: `order_tracking/{order_id}`
- Cart cleared after successful order creation
- Response includes order_id, order_number, status, total, tracking path

---

#### Task B-ORDER-02: Order Status State Machine
**Description**: Implement the order state machine: ordered → preparing → delivering → delivered (with cancellation from ordered/preparing). Each transition publishes `order.status_changed` event, updates Firebase RTDB, and triggers push notification.
**Dependencies**: B-ORDER-01
**Acceptance Criteria**:
- Valid transitions only: ordered→preparing→delivering→delivered, ordered→cancelled, preparing→cancelled
- Invalid transition returns 422
- Each transition records entry in order_timeline
- `order.status_changed` event published with old_status, new_status
- Firebase RTDB updated: `order_tracking/{id}/status/current`
- Push notification sent to customer on each status change
- Notification content matches translation keys from docs

---

#### Task B-ORDER-03: Order Cancellation & Refund
**Description**: Implement `POST /orders/{id}/cancel`. Support cancellation within policy window (before preparation starts: full refund; during preparation: vendor discretion). Process refund via payment gateway or mark cash order for manual refund.
**Dependencies**: B-ORDER-02, B-PAY-02
**Acceptance Criteria**:
- Cancellation before vendor acceptance: full refund, auto-cancel
- Cancellation after acceptance, before preparation: full refund (vendor can dispute)
- Cancellation during preparation: partial or no refund
- Refund processed via Stripe for card payments
- Cash refunds: wallet credit or manual process (MVP)
- Stock released on cancellation
- Customer notified of cancellation and refund status

---

#### Task B-ORDER-04: Order History & Reorder
**Description**: Implement `GET /orders` (paginated history) and `POST /orders/{id}/reorder`. Reorder checks item availability and adds available items to cart.
**Dependencies**: B-ORDER-01, B-CART-01
**Acceptance Criteria**:
- Order history paginated (20 per page), sorted by created_at DESC
- Each order includes vendor name, items summary, total, status, date
- Reorder checks each item availability at original vendor
- Available items added to cart
- Unavailable items shown in response with "no longer available" notice
- Customer can modify cart before checkout

---

### 5.7 Payment Module Tasks

---

#### Task B-PAY-01: Stripe Card Payment Integration
**Description**: Integrate Stripe for card tokenization and payment. Implement `POST /payments/tokenize/card` (client-side tokenization), `POST /payments/checkout` (server-side payment intent creation), and `POST /payments/webhook/stripe` (webhook handler). Support auth-then-capture flow.
**Dependencies**: B-AUTH-01
**Acceptance Criteria**:
- Card tokenization uses Stripe.js on client, server stores token reference
- Payment intent created with `capture_method: manual` for food orders
- 3D Secure handled via Stripe's automatic authentication
- Webhook processes: payment_intent.succeeded, payment_intent.failed
- Payment status tracked in `payments` table
- Card tokens stored for repeat payments (last_four, brand)
- PCI compliance: no raw card data touches our server

---

#### Task B-PAY-02: Cash on Delivery Support
**Description**: Implement cash payment method. No payment gateway interaction; order is confirmed immediately. Payment status set to 'pending_cash' until rider collects.
**Dependencies**: B-ORDER-01
**Acceptance Criteria**:
- Cash payment option available in payment methods
- Order created without payment authorization
- Payment record created with method='cash', status='pending_cash'
- Rider app shows cash collection amount
- Cash collection confirmed by rider updates payment status to 'captured'

---

#### Task B-PAY-03: Refund Processing
**Description**: Implement refund flow via Stripe. Support full and partial refunds. Update payment status and order total accordingly.
**Dependencies**: B-PAY-01
**Acceptance Criteria**:
- Full refund creates Stripe refund for full payment amount
- Partial refund creates Stripe refund for specified amount
- Payment status updated to 'refunded' (full) or 'partially_refunded'
- Refund processing time: 3-7 business days for card
- Refund event published for notification
- Cash order refund: wallet credit or manual processing

---

### 5.8 Dispatch Module Tasks

---

#### Task B-DISP-01: Rider Registration & Status
**Description**: Implement rider profile creation and status management. Riders can go online/offline, update current location. Location stored in Redis (`rider:{id}:location`) with 10-second TTL.
**Dependencies**: B-AUTH-01
**Acceptance Criteria**:
- Rider profile created with vehicle_type, phone_number
- Rider status: online, offline, on_delivery, on_break
- Location update API: `POST /rider/location` with lat, lng, heading, speed
- Location stored in Redis with 10-second TTL
- Location also pushed to Firebase RTDB for active assignments
- Rider only sees orders assigned to them

---

#### Task B-DISP-02: Order Assignment (Simple Dispatch)
**Description**: Implement basic dispatch algorithm. On `order.created` event: find online riders within 5km of vendor, score by proximity (primary) and rating (secondary), auto-assign to top candidate. If no rider available within 10 minutes, mark order as "dispatch_failed" and notify support.
**Dependencies**: B-DISP-01, B-ORDER-01
**Acceptance Criteria**:
- Dispatch triggered automatically on order confirmation
- Candidate riders filtered by: online status, within radius, not on_delivery (for MVP: 1 order at a time)
- Scoring: 70% proximity, 30% rating
- Auto-assign to top candidate (no broadcast for MVP)
- Assignment creates `delivery_assignments` record
- Assigned rider receives push notification
- Customer notified: "Rider assigned"
- Firebase RTDB updated with rider info
- Fallback: if no rider available in 10 min, notify operations team

---

#### Task B-DISP-03: Rider Order Lifecycle
**Description**: Implement rider-side order flow: accept assignment → navigate to vendor → confirm pickup → navigate to customer → confirm delivery. Each status change updates order status and Firebase RTDB.
**Dependencies**: B-DISP-02
**Acceptance Criteria**:
- Rider can accept/reject assignment (3 min timeout, then reassign)
- Pickup confirmation: `PATCH /rider/orders/{id}/status` with status='picked_up'
- Delivery confirmation: `PATCH /rider/orders/{id}/status` with status='delivered'
- GPS geofence validation: delivery only confirmed within 100m of delivery address
- Each status change triggers order status update (preparing → delivering → delivered)
- Each status change updates Firebase RTDB
- Push notification sent to customer at each stage

---

### 5.9 Search Module Tasks

---

#### Task B-SEARCH-01: Full-Text Search
**Description**: Implement `GET /search` using PostgreSQL full-text search with `pg_trgm` for fuzzy matching. Search across vendor names, cuisine types, and item names. Apply geo-filtering (within delivery radius). Support Arabic and English.
**Dependencies**: B-VENDOR-01
**Acceptance Criteria**:
- Search query matches vendor names, cuisine_types, and item names
- Results filtered by user's delivery area (geo-radius)
- Results filtered by is_open=true (configurable)
- Pagination support
- Sort by: relevance (default), delivery_time, rating, distance
- Arabic normalization: strip diacritics, normalize alef/ya
- Search latency target: < 300ms for 50-100 vendors

---

#### Task B-SEARCH-02: Autocomplete
**Description**: Implement `GET /search/autocomplete` with prefix matching. Return vendor names, item names, and popular queries. Cache in Redis with 15-minute TTL.
**Dependencies**: B-SEARCH-01
**Acceptance Criteria**:
- Returns up to 8 suggestions matching prefix
- Mix of vendor names, item names, and popular queries
- Popular queries derived from search analytics (top 100 queries)
- Results cached per prefix in Redis (15 min TTL)
- Response latency: < 50ms
- Supports Arabic and English prefixes

---

### 5.10 Notification Module Tasks

---

#### Task B-NOTIF-01: FCM Push Notifications
**Description**: Implement push notification delivery via FCM. Store device tokens per user. Route notifications based on type (transactional vs. marketing). Integrate with order status changes.
**Dependencies**: B-AUTH-01
**Acceptance Criteria**:
- Device token registered via `POST /notifications/device-token`
- Order status changes trigger push notification to customer
- Notification includes deep link to order tracking
- Transactional notifications use high-priority FCM delivery
- Failed deliveries retried once after 5 minutes
- Notification delivery tracked in notification_log table

---

#### Task B-NOTIF-02: Notification Preferences
**Description**: Implement `GET/PUT /notifications/preferences`. Allow users to opt out of marketing channels while keeping transactional notifications mandatory.
**Dependencies**: B-NOTIF-01
**Acceptance Criteria**:
- Default preferences: transactional ON, marketing ON
- Transactional notifications cannot be disabled
- Marketing notifications can be disabled
- Preferences stored per user in database
- Preference changes effective immediately

---

### 5.11 Infrastructure Tasks

---

#### Task B-INFRA-01: API Gateway Setup
**Description**: Set up reverse proxy (Nginx or Kong) with rate limiting, JWT validation, request logging, CORS, and SSL termination.
**Dependencies**: None
**Acceptance Criteria**:
- All API requests proxied through gateway
- Rate limiting: 100 req/min per user, 1000 req/min per IP
- JWT validation on protected endpoints
- CORS configured for mobile app origins
- SSL termination with TLS 1.2+
- Request timeout: 30s default, 60s for search
- Health check endpoint available

---

#### Task B-INFRA-02: Firebase RTDB Setup
**Description**: Configure Firebase project, set up RTDB structure for order tracking, configure security rules (customer can read own orders, rider can write location).
**Dependencies**: None
**Acceptance Criteria**:
- Firebase project created with RTDB enabled
- RTDB structure: `order_tracking/{order_id}/status`, `order_tracking/{order_id}/location`, `order_tracking/{order_id}/rider`
- Security rules: customer reads own order, rider writes location for assigned order
- Custom auth token generation for Firebase auth from backend
- Offline persistence enabled on client

---

#### Task B-INFRA-03: Internal Event Bus
**Description**: Implement a simple in-process event bus for asynchronous communication between modules. Support publish/subscribe pattern with at-least-once delivery. Events follow CloudEvents spec structure from `docs/09_backend_architecture.md`.
**Dependencies**: None
**Acceptance Criteria**:
- Events published with CloudEvents-compatible structure
- Subscribers receive events asynchronously (non-blocking)
- Event handlers execute in background (don't block API response)
- Events logged for debugging
- Failed event handlers logged and retried once
- Event types: order.created, order.status_changed, payment.captured, inventory.updated, rider.assigned

---

#### Task B-INFRA-04: Database Migrations & Seeding
**Description**: Set up database migration system (e.g., Prisma Migrate, Knex, or Alembic). Create seed data for 50-100 vendors with realistic menus (5-10 categories, 10-30 items each).
**Dependencies**: None
**Acceptance Criteria**:
- Migration system configured with up/down support
- All MVP tables created via migrations
- Seed script generates 50-100 vendors across different cuisines
- Each vendor has 5-10 menu categories with 10-30 items
- Items have options and choices where appropriate
- Vendor locations distributed across a real city (e.g., Dubai, Cairo)
- Seed data includes open/closed schedules

---

## 6. Module Dependency Graph

```
B-INFRA-01 ──┐
B-INFRA-02 ──┤
B-INFRA-03 ──┼── B-AUTH-01 ──┬── B-USER-01 ── B-USER-03
B-INFRA-04 ──┘               ├── B-USER-02 ── B-VENDOR-01
                              │                  ├── B-VENDOR-02 ── B-VENDOR-03
                              │                  │                  ├── B-INV-01
                              │                  │                  └── B-INV-02
                              │                  └── B-VENDOR-04
                              ├── B-CART-01 ── B-CART-02
                              │      │
                              │      ├── B-INV-01
                              │      └── B-ORDER-01 ── B-ORDER-02 ── B-ORDER-03
                              │            │
                              │            ├── B-PAY-01 ── B-PAY-02 ── B-PAY-03
                              │            │
                              │            └── B-DISP-01 ── B-DISP-02 ── B-DISP-03
                              │
                              ├── B-NOTIF-01 ── B-NOTIF-02
                              ├── B-PAY-01
                              └── B-DISP-01

B-VENDOR-01 ── B-SEARCH-01 ── B-SEARCH-02
```

---

## 7. Non-Functional Requirements

| Requirement | Target | Measurement |
|------------|--------|-------------|
| API response time (p99) | < 500ms | APM monitoring |
| Order placement latency | < 2s | End-to-end from tap to confirmation |
| Search latency (p95) | < 300ms | Query to result |
| Uptime | 99.5% | Monthly availability |
| Data consistency | Strong for orders/payments; eventual for search | Database transactions |
| Concurrent connections | 500 | Load test |
| Error rate | < 1% | Error tracking |
| Deployment time | < 15 minutes | CI/CD pipeline |
