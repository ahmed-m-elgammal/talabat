# MVP Overview — Talabat-Like Delivery Marketplace

## 1. Purpose

This document provides the master breakdown for a Talabat-inspired delivery marketplace MVP. It defines the minimum viable product scope, the boundaries between MVP (Phase 1) and medium-scale (Phase 2), and the organizational structure of the planning artifacts in the `MVP/` folder.

The target MVP must support at least **50-100 vendors/stores** with their full menu data, real-time inventory, ordering, payment, and delivery tracking. It does not replicate Talabat's full 9-country, multi-vertical, 15+ microservice production deployment — instead, it distills the essential delivery marketplace flows into a lean, implementable system.

---

## 2. MVP Definition

### 2.1 What the MVP Is

A single-country, single-vertical (Food + basic Grocery) delivery marketplace that enables the complete order lifecycle:

- **Browse & Discover**: Customers can search and browse vendors, view menus with real-time availability, and see personalized recommendations.
- **Order & Pay**: Customers can build a cart, apply vouchers, choose payment methods (card + cash), and place orders.
- **Track & Deliver**: Customers can track orders in real time, communicate with riders, and confirm delivery.
- **Vendor Management**: Vendors can manage menus, accept/reject orders, and update stock levels.
- **Rider Operations**: Riders can accept assignments, navigate to pickup/delivery, and update order status.

### 2.2 What the MVP Is NOT

- Multi-country deployment (one country only)
- Full fintech suite (wallet, BNPL, co-branded cards)
- Subscription/Pro membership system
- DineOut reservations
- Pharmacy with prescription flow
- Advanced AI-powered discovery (ChatGPT integration)
- AdTech/sponsored content
- Multi-search / photo-to-list
- Foldable device support
- HMS (Huawei Mobile Services) dual-platform support

### 2.3 Scale Target

| Dimension | MVP Target | Medium-Scale Target |
|-----------|-----------|-------------------|
| Vendors | 50-100 | 500-2,000 |
| Menu items | 2,000-5,000 | 50,000-200,000 |
| Concurrent users | 200-500 | 5,000-20,000 |
| Orders/day | 100-300 | 5,000-50,000 |
| Riders | 20-50 | 500-2,000 |
| Countries | 1 | 3-9 |

---

## 3. Architecture Decision: Simplified for MVP

The production Talabat uses 15+ microservices with Kafka, Istio service mesh, Kubernetes per country, and a multi-database strategy (PostgreSQL, MongoDB, Redis, Elasticsearch, Firebase RTDB). For the MVP, we adopt a **pragmatic monolith with modular boundaries** approach that preserves the domain separation but avoids the operational overhead of a full microservice deployment.

### 3.1 MVP Architecture: Modular Monolith

```
┌─────────────────────────────────────────────────────┐
│                    API Gateway                       │
│          (Rate limiting, JWT auth, routing)          │
└──────────────────────────┬──────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────┐
│              Modular Monolith Server                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐ │
│  │  User   │ │  Vendor │ │  Order  │ │ Payment  │ │
│  │ Module  │ │ Module  │ │ Module  │ │ Module   │ │
│  └─────────┘ └─────────┘ └─────────┘ └──────────┘ │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐              │
│  │Dispatch │ │Inventory│ │ Search  │              │
│  │ Module  │ │ Module  │ │ Module  │              │
│  └─────────┘ └─────────┘ └─────────┘              │
└──────────────────────────┬──────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────┐
│                    Data Layer                        │
│  ┌──────────┐ ┌────────┐ ┌───────────────────────┐ │
│  │PostgreSQL│ │ Redis  │ │ Firebase RTDB         │ │
│  │(Primary) │ │(Cache) │ │ (Real-time tracking)  │ │
│  └──────────┘ └────────┘ └───────────────────────┘ │
│  ┌──────────┐                                       │
│  │Elastic-  │  (Phase 2: add for search scale)     │
│  │search    │                                       │
│  └──────────┘                                       │
└─────────────────────────────────────────────────────┘
```

### 3.2 Why Modular Monolith for MVP

| Factor | Microservices (Production) | Modular Monolith (MVP) |
|--------|---------------------------|----------------------|
| Operational complexity | High (K8s, Istio, Kafka) | Low (single deployable) |
| Team size needed | 5-8 teams | 1-2 teams |
| Inter-service latency | Network hops + serialization | In-process calls |
| Data consistency | Eventual (Kafka) | Strong (transactional) |
| Deployment independence | Yes | No (deploy as one unit) |
| Scaling granularity | Per-service | Per-module (via horizontal scaling) |
| Migration path | — | Extract modules to services when needed |

### 3.3 Technology Stack for MVP

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Backend | Node.js (NestJS) or Python (FastAPI) | Rapid development, strong typing (NestJS), async support |
| Database | PostgreSQL 15+ | ACID compliance, JSONB for flexible fields, PostGIS for geo |
| Cache | Redis 7+ | Session, cart, stock status, rate limiting |
| Real-time | Firebase RTDB | Order tracking, rider location (same as production) |
| Search | PostgreSQL full-text + pg_trgm (MVP), Elasticsearch (Phase 2) | Sufficient for 50-100 vendors |
| Push notifications | FCM only (no HMS for MVP) | Google Play Services covers target market |
| Payments | Stripe (card) + Cash on Delivery | Simplified vs. production's dual-gateway |
| Mobile | Flutter 3.x | Same as production; cross-platform |
| Auth | JWT (RS256) | Same as production |
| CI/CD | GitHub Actions | Simple pipeline |
| Hosting | Single cloud (AWS/GCP) | No multi-region for MVP |

---

## 4. MVP Feature Breakdown

### 4.1 Core Features (Must-Have)

| # | Feature | User Story | Priority |
|---|---------|-----------|----------|
| F1 | Phone OTP authentication | As a customer, I want to sign up/login with my phone number | P0 |
| F2 | Guest browsing | As a visitor, I want to browse vendors without signing in | P0 |
| F3 | Vendor listing | As a customer, I want to see vendors near my address | P0 |
| F4 | Vendor menu | As a customer, I want to view a vendor's menu with prices | P0 |
| F5 | Item options & cart | As a customer, I want to add items with options to my cart | P0 |
| F6 | Address management | As a customer, I want to save delivery addresses | P0 |
| F7 | Checkout & payment | As a customer, I want to pay with card or cash | P0 |
| F8 | Order tracking | As a customer, I want to track my order in real time | P0 |
| F9 | Order history | As a customer, I want to view and reorder from past orders | P0 |
| F10 | Vendor order management | As a vendor, I want to accept/reject orders and update stock | P0 |
| F11 | Rider assignment | As a system, I want to assign riders to orders | P0 |
| F12 | Rider app basics | As a rider, I want to see assigned orders and navigate | P0 |
| F13 | Real-time rider location | As a customer, I want to see the rider's live position on a map | P0 |
| F14 | Push notifications | As a customer, I want to receive order status updates | P0 |
| F15 | Basic search | As a customer, I want to search for vendors and items | P0 |

### 4.2 Important Features (Should-Have for MVP+)

| # | Feature | User Story | Priority |
|---|---------|-----------|----------|
| F16 | Voucher/promo codes | As a customer, I want to apply discount vouchers | P1 |
| F17 | Favorites | As a customer, I want to save favorite vendors | P1 |
| F18 | Rider-customer chat | As a customer, I want to chat with my rider | P1 |
| F19 | Item availability status | As a customer, I want to see if items are in stock | P1 |
| F20 | Menu search | As a customer, I want to search within a vendor's menu | P1 |
| F21 | Reorder | As a customer, I want to quickly reorder a past order | P1 |
| F22 | Vendor ratings | As a customer, I want to rate and review vendors | P1 |
| F23 | Estimated delivery time | As a customer, I want to see accurate delivery ETAs | P1 |

### 4.3 Phase 2 Features (Medium-Scale)

| # | Feature | Notes |
|---|---------|-------|
| F24 | Multi-vertical (Grocery, Pharmacy) | Full grocery with finite stock, pharmacy with prescriptions |
| F25 | Wallet (talabat Pay) | Digital wallet with top-up |
| F26 | BNPL (Buy Now Pay Later) | PostPaid installments |
| F27 | Pro subscription | Free delivery, exclusive offers |
| F28 | DineOut reservations | Capacity-based inventory |
| F29 | AI chat (ChatGPT) | Conversational food discovery |
| F30 | Multi-country | Separate deployments per country |
| F31 | Elasticsearch | Full-text search at scale |
| F32 | Microservice extraction | Split modules into independent services |
| F33 | HMS push support | Huawei device notifications |
| F34 | Advanced dispatch | ML-based demand prediction, batch dispatch |
| F35 | AdTech / sponsored content | Monetization through ads |

---

## 5. Planning Document Structure

Each file in the `MVP/` folder covers one plan area:

| File | Scope | Key Contents |
|------|-------|-------------|
| `00_MVP_Overview.md` | This document | MVP definition, architecture, feature breakdown |
| `01_Backend_MVP_Scope.md` | Backend services | API design, database schema, service modules, tasks |
| `02_Frontend_MVP_Scope.md` | Flutter app | Feature modules, state management, navigation, tasks |
| `03_UI_UX_Scope.md` | Design system & screens | Components, screen specs, accessibility, tasks |
| `04_Budget_Cost_Efficiency.md` | Cost planning | Infrastructure costs, team sizing, efficiency strategies |
| `05_Realtime_Architecture.md` | Real-time considerations | Tracking, push, chat, offline, performance |
| `06_Medium_Scale_Roadmap.md` | Phase 2 path | Migration from monolith to microservices, scaling plan |

---

## 6. Guiding Principles

1. **Ship the core loop first**: Browse → Cart → Pay → Track → Deliver. Everything else is secondary.
2. **Preserve domain boundaries**: Even in a monolith, keep module interfaces clean so extraction to microservices is straightforward.
3. **Design for 50-100 vendors, not 50,000**: Optimize for correctness and developer velocity, not peak Ramadan traffic.
4. **Real-time is non-negotiable**: Order tracking and status updates must be real-time from day one. This is a core differentiator.
5. **Offline-first on the client**: The mobile app must handle network transitions gracefully, matching the production app's offline resilience.
6. **One country, one currency, one language pair**: English + Arabic (LTR/RTL) for the MVP. Multi-country and dialects are Phase 2.
7. **PCI compliance from day one**: Use Stripe tokenization; never touch raw card data.
8. **Test the real-time path early**: Firebase RTDB integration should be in the first sprint, not the last.

---

## 7. Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Scope creep (adding Pro/BNPL to MVP) | High | Strict feature list with P0/P1/P2 classification |
| Real-time latency issues | High | Early load testing of Firebase RTDB with simulated riders |
| Flutter performance on low-end devices | Medium | Performance budget per screen (TTI targets from docs) |
| Payment integration delays | Medium | Use Stripe test mode first; cash-on-delivery as fallback |
| Rider app complexity | Medium | Simplify rider app to essential flows only |
| Arabic/RTL layout bugs | Medium | RTL testing from sprint 1; use production's RTL patterns |
