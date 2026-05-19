# 06 — Medium-Scale Roadmap (Phase 2 Path)

## Overview

This document maps the journey from MVP to medium-scale, targeting 500+ vendors, 10,000+ daily orders, and multiple verticals across 3+ countries. Each phase builds on the previous one, and the modular monolith design ensures that service boundaries are already defined in code, making extraction straightforward when scale demands it.

---

## Phase Timeline

| Phase | Timeline | Key Milestone |
|-------|----------|---------------|
| MVP | Week 1–12 | Food delivery in 1 country, 50–100 vendors |
| Phase 2A | Month 4–6 | Scale to 500 vendors, add search, monitoring |
| Phase 2B | Month 7–9 | Add grocery vertical, multi-payment, subscriptions |
| Phase 2C | Month 10–12 | Multi-country, extract microservices, advanced real-time |

---

## Phase 2A: Scale & Observability (Month 4–6)

### T6.1 — Elasticsearch Search Integration

**Description:** Replace PostgreSQL full-text search with Elasticsearch for Arabic/English tokenization, autocomplete, personalized ranking, and geo-aware search. This is the highest-priority Phase 2 item because search quality directly impacts order conversion.

**Dependencies:** MVP complete with vendor data flowing.

**Acceptance Criteria:**
- [ ] Elasticsearch cluster deployed (3-node, managed)
- [ ] `vendor_index` per country with Arabic and English analyzers
- [ ] Diacritics stripping and Arabic letter normalization (أ→ا, ة→ه)
- [ ] Autocomplete endpoint: GET /search/autocomplete?q={prefix}
- [ ] Search endpoint with multi-field matching, geo filtering, and personalization boost
- [ ] Index sync: vendor/menu updates in PostgreSQL → Elasticsearch via change data capture
- [ ] Search latency target: < 200ms (p95)

**Phase:** Phase 2A

---

### T6.2 — Monitoring & Observability

**Description:** Add application performance monitoring, error tracking, and infrastructure metrics. Essential before scaling further.

**Dependencies:** MVP deployed in production.

**Acceptance Criteria:**
- [ ] Sentry for error tracking in Flutter app and backend
- [ ] Structured logging (JSON) with correlation IDs across all backend modules
- [ ] Prometheus + Grafana for infrastructure metrics (CPU, memory, request latency, error rate)
- [ ] Health check dashboards for all services
- [ ] Alerting: p99 latency > 2s, error rate > 1%, database connections > 80%

**Phase:** Phase 2A

---

### T6.3 — Kubernetes Migration

**Description:** Move from single VM to Kubernetes (EKS or GKE) with auto-scaling, rolling deployments, and health checks. This enables horizontal scaling for order peaks.

**Dependencies:** MVP running on VM with Docker.

**Acceptance Criteria:**
- [ ] Backend deployed as Docker container on K8s
- [ ] Horizontal Pod Autoscaler: CPU > 70% → scale up (3–10 replicas)
- [ ] Rolling deployments with zero downtime
- [ ] Liveness and readiness probes configured
- [ ] PostgreSQL and Redis remain managed services (not in K8s for reliability)
- [ ] Staging environment on K8s mirrors production

**Phase:** Phase 2A

---

### T6.4 — Feature Flag Service

**Description:** Build a lightweight feature flag service supporting boolean flags, percentage rollouts, and country-level targeting. Replaces environment variables with a dynamic system.

**Dependencies:** MVP running.

**Acceptance Criteria:**
- [ ] Feature flags stored in PostgreSQL with Redis cache (5-min TTL)
- [ ] Admin UI to create/toggle flags
- [ ] Client SDK: evaluate flag value given user context (country, user segment, device)
- [ ] Kill switch support: instantly disable features in production
- [ ] A/B experiment support: percentage bucket assignment

**Phase:** Phase 2A

---

## Phase 2B: Verticals & Payments (Month 7–9)

### T6.5 — Grocery Vertical (Q-Commerce)

**Description:** Add grocery delivery with finite stock inventory, dark store model, and item replacement flow. This is the most complex Phase 2 feature because it introduces a fundamentally different inventory model.

**Dependencies:** T6.1 (search for grocery product search), Inventory Service (T1.P2.1).

**Acceptance Criteria:**
- [ ] Inventory Service with finite stock: stock_count, low_stock_threshold, out_of_stock_behavior
- [ ] Stock reservation on cart add (10-min TTL), deduction on order confirm
- [ ] Item replacement timer: 5-minute countdown when item out of stock during picking
- [ ] Grocery-specific search: product names, brand names, category browsing
- [ ] Category-based product listing with filters (price, brand, dietary)
- [ ] Age-restricted item confirmation dialog
- [ ] "Few left" badge for low-stock items

**Phase:** Phase 2B

---

### T6.6 — Multi-Payment Integration

**Description:** Add Apple Pay, Google Pay, and talabat Pay wallet. Add a second payment gateway (HyperPay) for regional methods (STC Pay, BenefitPay, etc.).

**Dependencies:** MVP payment service with single gateway.

**Acceptance Criteria:**
- [ ] Apple Pay integration via native PassKit framework
- [ ] Google Pay integration via Google Pay API
- [ ] Wallet service: balance, top-up, transactions, transfer
- [ ] HyperPay integration for regional payment methods
- [ ] Gateway routing: token affinity (use same gateway as original tokenization)
- [ ] 3D Secure 2 support for both gateways

**Phase:** Phase 2B

---

### T6.7 — Subscription Service (talabat Pro)

**Description:** Add Pro subscription with free delivery on eligible vendors, exclusive offers, and family plans. Drives customer retention and order frequency.

**Dependencies:** T6.6 (wallet for payment), T1.2 (vendor pro_eligible flag).

**Acceptance Criteria:**
- [ ] Plan management: Pro and Pro Lite tiers with monthly fees
- [ ] Free delivery on pro_eligible vendors (delivery fee = $0 in pricing calculation)
- [ ] Subscription signup with payment method selection
- [ ] Auto-renewal with retry on payment failure
- [ ] Family plan: add members by phone number (up to 4)
- [ ] Cancellation flow with reason survey
- [ ] Pro badge on vendor cards and in profile

**Phase:** Phase 2B

---

### T6.8 — Notification Service (Braze Integration)

**Description:** Add Braze for marketing automation, lifecycle campaigns, and in-app messages. Replaces simple FCM pushes with a sophisticated engagement platform.

**Dependencies:** MVP FCM push running.

**Acceptance Criteria:**
- [ ] Braze SDK integrated in Flutter app
- [ ] 12 notification channels matching full architecture spec
- [ ] Transactional pushes via FCM (as before)
- [ ] Marketing campaigns via Braze: segmentation, A/B testing, scheduling
- [ ] In-app messages: full-screen modal, slide-up, custom HTML
- [ ] Smart suppression: daily cap (5 marketing pushes/24h), sleep hours (11 PM–8 AM)
- [ ] User attributes synced to Braze: first_name, country, pro_status, last_order_date

**Phase:** Phase 2B

---

## Phase 2C: Multi-Country & Microservices (Month 10–12)

### T6.9 — Multi-Country Deployment

**Description:** Expand to 2–3 additional countries with data isolation, per-country database partitions, and geoDNS routing.

**Dependencies:** T6.3 (Kubernetes for multi-cluster).

**Acceptance Criteria:**
- [ ] Country-level data isolation: separate PostgreSQL schemas or databases per country
- [ ] Country code in all API requests (`X-Country-Code` header)
- [ ] Per-country vendor pools, riders, and menus
- [ ] Currency support: AED (UAE), SAR (Saudi), EGP (Egypt) — configurable per country
- [ ] GeoDNS: route users to nearest country backend
- [ ] Cross-country SSO: authenticate once, access any country

**Phase:** Phase 2C

---

### T6.10 — Microservice Extraction

**Description:** Extract the most performance-critical service modules from the monolith into independent microservices. Priority order: Order Service, Dispatch Service, Payment Service, then others.

**Dependencies:** T6.3 (Kubernetes), T6.11 (Kafka event bus).

**Acceptance Criteria:**
- [ ] Order Service extracted: independent deployment, own database connection
- [ ] Dispatch Service extracted: handles rider assignment independently
- [ ] Payment Service extracted: isolated for PCI-DSS compliance boundary
- [ ] Inter-service communication via Kafka events and REST/gRPC
- [ ] Service mesh (Istio) for mTLS between services
- [ ] Each service has own CI/CD pipeline

**Phase:** Phase 2C

---

### T6.11 — Kafka Event Bus

**Description:** Introduce Apache Kafka for asynchronous inter-service communication. Enables event-driven workflows, event replay, and decoupled services.

**Dependencies:** T6.3 (Kubernetes for Kafka deployment).

**Acceptance Criteria:**
- [ ] Managed Kafka (MSK or Confluent) deployed
- [ ] Event topics: order.created, order.status_changed, payment.captured, inventory.updated, rider.location_updated
- [ ] Events follow CloudEvents specification for standardized metadata
- [ ] Existing in-process events migrated to Kafka publishers/consumers
- [ ] Consumer lag monitoring and auto-scaling
- [ ] Event schema registry for backward compatibility

**Phase:** Phase 2C

---

### T6.12 — Advanced Real-Time

**Description:** Upgrade real-time architecture with WebSocket migration, SSE for menu availability, and ML-based ETA engine.

**Dependencies:** T6.11 (Kafka for event streaming), T6.10 (extracted dispatch service).

**Acceptance Criteria:**
- [ ] WebSocket server for order tracking (replacing Firebase RTDB for high-volume paths)
- [ ] SSE for real-time menu availability updates while browsing
- [ ] ML-based ETA engine: combines historical data, traffic, weather, order complexity
- [ ] Android Live Activity: foreground service for persistent tracking notification
- [ ] iOS Live Activity: Dynamic Island and Lock Screen widgets
- [ ] Advanced chat: image sharing, predefined messages, support agent handoff

**Phase:** Phase 2C

---

### T6.13 — Rewards & BNPL

**Description:** Add loyalty points system and Buy Now Pay Later (PostPaid) to increase customer retention and order value.

**Dependencies:** T6.6 (wallet infrastructure), T6.7 (subscription for Pro integration).

**Acceptance Criteria:**
- [ ] Rewards: points earn on orders, burn options (free delivery, money off, charity, raffle)
- [ ] Points expiration and balance tracking
- [ ] BNPL: PostPaid with 30-day payment cycle
- [ ] BNPL dashboard: available balance, upcoming payments, overdue alerts
- [ ] Rewind: retroactively convert paid orders to BNPL
- [ ] Multi-order BNPL payment: pay multiple installments at once
- [ ] BNPL credit limit management

**Phase:** Phase 2C

---

## Dependency Graph (Phase 2)

```
Phase 2A (Month 4-6):
  T6.1 Elasticsearch  ←── requires MVP vendor data
  T6.2 Monitoring     ←── requires MVP in production
  T6.3 Kubernetes     ←── requires MVP Docker-ized
  T6.4 Feature Flags  ←── requires MVP running

Phase 2B (Month 7-9):
  T6.5 Grocery        ←── requires T6.1 (search) + T1.P2.1 (inventory)
  T6.6 Multi-Payment  ←── requires MVP payment service
  T6.7 Subscriptions  ←── requires T6.6 (wallet)
  T6.8 Braze          ←── requires MVP FCM

Phase 2C (Month 10-12):
  T6.9 Multi-Country  ←── requires T6.3 (K8s)
  T6.10 Microservices ←── requires T6.3 (K8s) + T6.11 (Kafka)
  T6.11 Kafka         ←── requires T6.3 (K8s)
  T6.12 Adv Real-Time ←── requires T6.10 + T6.11
  T6.13 Rewards/BNPL  ←── requires T6.6 + T6.7
```

---

## Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Premature microservice extraction | High operational complexity, slower development | Stay on monolith until K8s + Kafka are ready; extract one service at a time |
| Elasticsearch cost overruns | Higher infrastructure bill | Start with single-node ES; scale only when query volume demands it |
| Multi-country data compliance | Legal/regulatory issues | Consult local counsel before launching in each country; implement data residency from day 1 |
| Braze cost scaling with MAU | Unexpected marketing costs | Monitor Braze usage closely; set campaign limits; evaluate open-source alternatives if costs spike |
| Kafka operational complexity | Team skill gap, reliability issues | Use managed Kafka (MSK/Confluent); invest in team training before migration |
