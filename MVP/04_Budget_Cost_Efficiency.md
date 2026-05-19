# 04 — Budget & Cost Efficiency

## Overview

This plan outlines the cost structure for building and running the Talabat MVP across infrastructure, third-party services, and team resources. The MVP is designed for cost efficiency: a modular monolith on a single server, a single payment gateway, Firebase's free tier for real-time features, and a small focused team. The goal is to reach market validation with minimal spend while maintaining a clear upgrade path for scale.

---

## Infrastructure Costs

### T4.1 — Compute & Hosting (Monthly)

**Description:** The MVP backend runs as a single monolith on a cloud VM. A single PostgreSQL instance and a single Redis instance handle all data needs. No Kubernetes, no multi-region, no load balancers for MVP.

**Dependencies:** None.

**Acceptance Criteria:**
- [ ] Single cloud VM (4 vCPU, 8GB RAM) — estimated $80–120/month (AWS t3.xlarge or equivalent)
- [ ] PostgreSQL managed instance (db.t3.medium) — estimated $50–80/month
- [ ] Redis managed instance (cache.t3.small) — estimated $20–30/month
- [ ] Total compute: ~$150–230/month
- [ ] Auto-scaling NOT configured for MVP (manual scale-up if needed)
- [ ] Single availability zone deployment (no cross-AZ redundancy for MVP)

**Phase:** MVP

**Cost Estimate:** $150–230/month

---

### T4.2 — Firebase Realtime Database (Monthly)

**Description:** Firebase RTDB handles live order tracking (rider GPS, order status) and rider-customer chat. The MVP uses the Flame plan ($25/month) which covers 1GB stored and 10GB/month downloaded — sufficient for 100–500 daily orders.

**Dependencies:** None.

**Acceptance Criteria:**
- [ ] Firebase project with RTDB enabled
- [ ] Security rules enforce user-specific access (customers can only read their own orders)
- [ ] Data volume stays within Flame plan limits (< 10GB/month download for MVP scale)
- [ ] Monitor usage dashboard for overages

**Phase:** MVP

**Cost Estimate:** $25/month (Flame plan)

---

### T4.3 — Payment Gateway Fees (Per Transaction)

**Description:** Payment processing costs are transaction-based. Using Stripe or Checkout.com for card payments. Cash on delivery has no gateway fee but has operational costs for cash handling.

**Dependencies:** T1.4 (payment service).

**Acceptance Criteria:**
- [ ] Stripe/Checkout.com pricing: 2.9% + 30¢ per card transaction (standard MENA rates may vary)
- [ ] No monthly minimum or setup fee for MVP
- [ ] Cash on delivery: no gateway fee but factor in ~2% cash handling cost
- [ ] Target 70% card / 30% cash split based on MENA market data

**Phase:** MVP

**Cost Estimate:** ~3% of card transaction volume

---

### T4.4 — Push Notifications (Monthly)

**Description:** FCM (Firebase Cloud Messaging) for push notifications is free for unlimited messages. No need for Braze engagement platform in MVP — simple transactional pushes only.

**Dependencies:** None.

**Acceptance Criteria:**
- [ ] FCM configured for Android and iOS
- [ ] Transactional pushes only: order status updates, delivery updates
- [ ] No marketing/engagement push campaigns in MVP
- [ ] No third-party push service (Braze, OneSignal) needed for MVP

**Phase:** MVP

**Cost Estimate:** $0/month (FCM is free)

---

### T4.5 — SMS/OTP Costs (Monthly)

**Description:** OTP delivery via SMS. Cost depends on volume and provider. Using a regional SMS provider (e.g., Twilio, MessageBird) with MENA coverage.

**Dependencies:** T1.1 (user service).

**Acceptance Criteria:**
- [ ] SMS cost per OTP: ~$0.02–0.05 per message (varies by country)
- [ ] Rate limiting (3 OTPs per 5 minutes) controls cost
- [ ] Estimated volume: 500 OTPs/month (500 new registrations + password resets)
- [ ] Monthly cost: ~$10–25/month

**Phase:** MVP

**Cost Estimate:** $10–25/month

---

### T4.6 — CDN & Storage (Monthly)

**Description:** Static asset delivery (vendor logos, menu item images) via CDN. Image storage on S3-compatible object storage.

**Dependencies:** T1.2 (vendor service for image uploads).

**Acceptance Criteria:**
- [ ] S3-compatible storage for vendor logos and menu images (~5GB for 100 vendors)
- [ ] CDN (CloudFront or equivalent) for image delivery
- [ ] Cost: ~$1–5/month for storage + $5–15/month for CDN bandwidth
- [ ] Image optimization: server-side resize and WebP conversion to reduce bandwidth

**Phase:** MVP

**Cost Estimate:** $6–20/month

---

### T4.7 — App Store Costs (Annual)

**Description:** Developer accounts for publishing the mobile app.

**Dependencies:** None.

**Acceptance Criteria:**
- [ ] Apple Developer Program: $99/year
- [ ] Google Play Developer Account: $25 one-time
- [ ] Total: ~$124 first year, $99/year ongoing

**Phase:** MVP

**Cost Estimate:** $124 first year

---

## Total MVP Monthly Cost Summary

| Category | Monthly Cost |
|----------|-------------|
| Compute (VM + DB + Redis) | $150–230 |
| Firebase RTDB | $25 |
| SMS/OTP | $10–25 |
| CDN & Storage | $6–20 |
| Push Notifications | $0 |
| **Subtotal (fixed)** | **$191–300/month** |
| Payment Gateway | ~3% of card volume |
| App Store | ~$10/month (annualized) |

**Estimated total fixed cost: ~$200–310/month**

At 100 daily orders with average order value (AOV) of AED 50 (~$13.60), monthly GMV ≈ $40,800. At 3% payment fee on 70% card transactions, payment costs ≈ $857/month. **Total operational cost: ~$1,060–1,170/month.**

---

## Team Budget

### T4.8 — MVP Development Team

**Description:** The minimum team needed to build and launch the MVP in 8–12 weeks. Small, cross-functional team with overlap between roles.

**Dependencies:** None.

**Acceptance Criteria:**
- [ ] 1 Backend Developer (Node.js/Python + PostgreSQL + Redis) — full-time for 8–12 weeks
- [ ] 1 Flutter Developer — full-time for 8–12 weeks
- [ ] 1 UI/UX Designer — part-time (50%) for first 4 weeks (design system + screen designs), then advisory
- [ ] 1 Product Manager / QA — part-time (50%) for requirements clarification and acceptance testing
- [ ] No dedicated DevOps for MVP (backend dev handles deployment)
- [ ] No dedicated iOS/Android native developers (Flutter handles both platforms)

**Phase:** MVP

**Cost Estimate:** Depends on market; estimated 2.5 FTE × 10 weeks

---

## Phase 2 Budget Items (Not for MVP)

### T4.P2.1 — Kubernetes Migration
**Description:** Move from single VM to Kubernetes (EKS/GKE) with auto-scaling, multi-AZ, and service mesh. Cost increases to ~$500–1,500/month.
**Phase:** Phase 2

### T4.P2.2 — Elasticsearch Cluster
**Description:** Add Elasticsearch for full-text search with Arabic/English tokenization. Cost: ~$100–300/month for a small cluster.
**Phase:** Phase 2

### T4.P2.3 — Braze Engagement Platform
**Description:** Marketing automation, lifecycle campaigns, in-app messages. Cost: $5,000–15,000/month depending on MAU.
**Phase:** Phase 2

### T4.P2.4 — Multi-Country Infrastructure
**Description:** Per-country database clusters, geoDNS, and CDN PoPs. Multiplies compute costs by number of countries.
**Phase:** Phase 2

### T4.P2.5 — Kafka Event Streaming
**Description:** Managed Kafka (Confluent/MSK) for event-driven architecture. Cost: ~$300–500/month for a small cluster.
**Phase:** Phase 2

### T4.P2.6 — Monitoring & Observability Stack
**Description:** New Relic/Sentry for APM, Prometheus/Grafana for infrastructure metrics, PagerDuty for alerting. Cost: $200–500/month.
**Phase:** Phase 2

### T4.P2.7 — Additional Payment Gateways
**Description:** HyperPay for regional methods, Apple Pay/Google Pay integration, wallet infrastructure. Integration costs + per-transaction fees.
**Phase:** Phase 2

### T4.P2.8 — Team Scaling
**Description:** Add DevOps engineer, dedicated QA, second backend dev, second Flutter dev, product designer. Team grows from 2.5 FTE to 6–8 FTE.
**Phase:** Phase 2
