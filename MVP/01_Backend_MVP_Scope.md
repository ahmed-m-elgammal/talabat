# 01 — Backend MVP Scope

## Overview

The MVP backend implements a **modular monolith** that mirrors Talabat's microservice boundaries but deploys as a single application. This reduces operational complexity while preserving clean domain separation for future extraction. The backend serves a single country with food delivery only, supporting 50–100 vendors and 100–500 daily orders.

The full Talabat backend (see `docs/09_backend_architecture.md`) runs 15 microservices across Kubernetes clusters with Kafka event streaming, Elasticsearch search, and multi-country deployments. The MVP strips this down to 5 core service modules communicating via in-process function calls and a lightweight event bus, with PostgreSQL as the sole persistent store and Redis for caching.

---

## Service Modules

### T1.1 — User Service Module

**Description:** Handles user registration, authentication via OTP, profile management, and address book. Implements JWT-based session management with access/refresh token pairs. Supports guest browsing without authentication. This module owns the `users`, `user_auth_providers`, and `addresses` database tables.

**Dependencies:** None (foundational module).

**Acceptance Criteria:**
- [ ] POST /auth/otp/send sends a 6-digit OTP to a mobile number with rate limiting (3 per 5 minutes)
- [ ] POST /auth/otp/verify validates OTP and returns JWT access token (1h expiry) + refresh token (30d expiry)
- [ ] GET /users/me returns authenticated user profile
- [ ] PATCH /users/me updates profile fields (name, email, DOB, gender)
- [ ] GET /users/me/addresses returns saved addresses
- [ ] POST /users/me/addresses creates a new address with geohash
- [ ] JWT tokens are RS256-signed with key rotation support
- [ ] Guest mode allows unauthenticated browsing of vendors and menus

**Phase:** MVP

---

### T1.2 — Vendor Service Module

**Description:** Manages restaurant/vendor CRUD, menu management (categories, items, options, choices), operating hours, and area coverage. Vendors are filtered by delivery radius from the customer's address. The menu model supports the three-level hierarchy (Category → Item → Options → Choices) described in `docs/05_inventory_system.md`. For the MVP, food items use the **infinite stock model** (availability is determined by vendor open/closed status, not stock counts).

**Dependencies:** T1.1 (user authentication for vendor portal access).

**Acceptance Criteria:**
- [ ] GET /vendors returns vendors filtered by lat/lng within delivery radius, paginated (20 per page)
- [ ] GET /vendors/{id}/menu returns full menu with categories, items, options, and choices
- [ ] Menu response includes ETag for conditional GET (304 Not Modified when unchanged)
- [ ] Vendor listing supports basic filters: cuisine type, open status, sort by (recommended, delivery_time, rating)
- [ ] Vendor operating hours are enforced: items from closed vendors are unavailable
- [ ] Admin endpoint to create/update vendor profiles and menus

**Phase:** MVP

---

### T1.3 — Order Service Module

**Description:** Orchestrates the complete order lifecycle from cart through delivery confirmation. Implements single-vendor cart (adding items from a different vendor clears the cart). Manages the order state machine: ORDERED → PREPARING → DELIVERING → DELIVERED, with cancellation branches. Handles order pricing calculation (subtotal, delivery fee, service fee, total). Integrates with the inventory module for availability checks at order time.

**Dependencies:** T1.1 (user auth), T1.2 (vendor/menu data), T1.4 (payment), T1.5 (dispatch).

**Acceptance Criteria:**
- [ ] POST /cart/items adds item to cart; fails if item unavailable or vendor differs from current cart
- [ ] DELETE /cart/items/{id} removes item; DELETE /cart clears entire cart
- [ ] POST /cart/voucher applies voucher code with server-side validation
- [ ] POST /orders creates order from cart, validates item availability, reserves stock, triggers payment
- [ ] GET /orders/{id} returns order with status, items, pricing, rider info, and timeline
- [ ] GET /orders returns paginated order history
- [ ] POST /orders/{id}/cancel cancels order within policy window (before vendor accepts = full refund)
- [ ] Order state transitions are logged with timestamps in order timeline
- [ ] Pricing calculation includes: subtotal, delivery fee (flat rate per vendor), service fee (5%), total

**Phase:** MVP

---

### T1.4 — Payment Service Module

**Description:** Handles payment authorization, capture, and refund through a single payment gateway. For MVP, supports credit/debit card (via Stripe or Checkout.com tokenization) and cash on delivery. Card tokenization is handled client-side; the backend stores only token references (PCI-DSS compliance). Payment authorization happens at order creation; capture happens at delivery confirmation. Refunds are processed for cancellations within policy.

**Dependencies:** T1.1 (user auth for saved payment methods).

**Acceptance Criteria:**
- [ ] POST /payments/methods returns saved payment methods for user (card tokens + cash option)
- [ ] POST /payments/tokenize/card stores card token reference (not raw card data)
- [ ] Payment authorization at order creation with 15-minute timeout
- [ ] Payment capture triggered on delivery confirmation
- [ ] POST /orders/{id}/refund processes refund to original payment method for valid cancellations
- [ ] Cash on delivery orders skip authorization step
- [ ] Payment status tracked: initiated → authorized → captured → refunded

**Phase:** MVP

---

### T1.5 — Dispatch Service Module

**Description:** Manages rider assignment and basic delivery tracking. For MVP, uses a simplified dispatching algorithm: when an order is confirmed, find the nearest available rider and assign. Supports manual rider management (admin adds/removes riders). Rider location is updated via the rider app and stored in Firebase RTDB for real-time customer tracking.

**Dependencies:** T1.3 (order service triggers dispatch).

**Acceptance Criteria:**
- [ ] Rider CRUD: admin can register riders with name, phone, vehicle type, coverage zone
- [ ] POST /dispatch/assign assigns nearest available rider to a confirmed order
- [ ] Rider status management: online, offline, on_delivery
- [ ] Delivery confirmation: rider marks order as picked up, then delivered
- [ ] Vendor Own Delivery (VOD) option: vendors with self-delivery skip dispatch assignment

**Phase:** MVP

---

### T1.6 — API Gateway & Middleware

**Description:** Sets up the HTTP server with middleware for authentication, rate limiting, CORS, request logging, and error handling. All routes are versioned (`/v1/`). This is a lightweight API gateway since we are deploying a monolith — no separate gateway service needed. Implements the same header conventions as the full architecture: `Authorization`, `X-Device-ID`, `X-Correlation-ID`.

**Dependencies:** None (foundational infrastructure).

**Acceptance Criteria:**
- [ ] All routes under `/v1/` prefix
- [ ] JWT validation middleware on protected routes; public routes for vendor listing and menu browsing
- [ ] Rate limiting: 100 req/min per user, 10 req/min for auth endpoints
- [ ] Structured JSON error responses with error code, message, and request_id
- [ ] CORS configured for web and mobile client origins
- [ ] Request/response logging with correlation ID propagation
- [ ] Health check endpoint: GET /health returns 200

**Phase:** MVP

---

### T1.7 — Database Schema & Migrations

**Description:** Creates the PostgreSQL database schema covering all MVP entities: users, addresses, vendors, vendor_schedules, menu_categories, menu_items, item_options, option_choices, orders, order_items, payments, payment_tokens, delivery_assignments, riders, vouchers. Implements database migrations with version tracking. Seeds initial data for 50–100 vendors with realistic menus.

**Dependencies:** None (foundational infrastructure).

**Acceptance Criteria:**
- [ ] Migration system with up/down support
- [ ] All tables use UUID primary keys
- [ ] Indexes on: vendors(country_code, is_open), orders(user_id, status, created_at), menu_items(vendor_id, is_available)
- [ ] Geospatial index on vendors(latitude, longitude) using PostGIS for radius queries
- [ ] Seed script creates 50–100 vendors with 3–8 categories each, 5–15 items per category
- [ ] Foreign key constraints enforce referential integrity
- [ ] Migration rollback is safe and tested

**Phase:** MVP

---

### T1.8 — Redis Caching Layer

**Description:** Implements Redis caching for frequently accessed data: vendor listing by area, menu data, feature flags, and session data. Uses the same key patterns as the full architecture (e.g., `vendor:{id}:status`, `cart:{user_id}:{vendor_id}`) but with a single Redis instance instead of a cluster.

**Dependencies:** T1.7 (database schema must exist first).

**Acceptance Criteria:**
- [ ] Vendor listing cached by area with 5-minute TTL
- [ ] Menu content hash cached for ETag validation
- [ ] Server-side cart cached with 24-hour TTL
- [ ] OTP rate limiting via Redis key with expiry
- [ ] Cache invalidation on vendor/menu updates
- [ ] Graceful degradation: if Redis is unavailable, queries fall through to PostgreSQL

**Phase:** MVP

---

## Phase 2 Backend Tasks (Not for MVP)

### T1.P2.1 — Inventory Service (Finite Stock)
**Description:** Add finite stock tracking for grocery/pharmacy verticals with reservation and deduction phases. Currently MVP uses infinite stock model (food only).
**Phase:** Phase 2

### T1.P2.2 — Search Service (Elasticsearch)
**Description:** Replace PostgreSQL full-text search with Elasticsearch for Arabic/English tokenization, autocomplete, and personalized ranking. MVP uses PostgreSQL `tsvector` with basic text matching.
**Phase:** Phase 2

### T1.P2.3 — Wallet & BNPL Services
**Description:** Add talabat Pay digital wallet and PostPaid BNPL system with installment management. MVP supports only card + cash.
**Phase:** Phase 2

### T1.P2.4 — Subscription Service (talabat Pro)
**Description:** Add Pro subscription with free delivery, exclusive offers, and family plans. MVP has no subscription tier.
**Phase:** Phase 2

### T1.P2.5 — Rewards Service
**Description:** Loyalty points system with earn/spend/expire mechanics. MVP has no rewards program.
**Phase:** Phase 2

### T1.P2.6 — Notification Service (Braze Integration)
**Description:** Multi-channel notification with Braze for marketing campaigns, lifecycle engagement, and smart suppression. MVP uses simple FCM push for transactional notifications only.
**Phase:** Phase 2

### T1.P2.7 — Kafka Event Bus
**Description:** Replace in-process event dispatch with Apache Kafka for async inter-service communication. Enables event-driven workflows at scale.
**Phase:** Phase 2

### T1.P2.8 — Multi-Country Deployment
**Description:** Add country-level data isolation, per-country database sharding, and geoDNS routing. MVP is single-country only.
**Phase:** Phase 2

### T1.P2.9 — Feature Flag Service
**Description:** Server-side feature flag evaluation with percentage rollouts, A/B experiments, and kill switches. MVP uses simple environment variables for feature flags.
**Phase:** Phase 2

### T1.P2.10 — Fraud Prevention Pipeline
**Description:** reCAPTCHA Enterprise, device fingerprinting (Shield Service), location verification (Incognia), and VM detection. MVP uses basic rate limiting only.
**Phase:** Phase 2
