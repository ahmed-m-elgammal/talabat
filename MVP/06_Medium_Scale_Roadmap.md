# 06 — Medium-Scale Roadmap (Phase 2)

## 1. Overview

This document outlines the path from MVP to medium-scale operations, where the platform handles **500-2,000 vendors, 5,000-50,000 daily orders, and 3-9 country deployments**. The roadmap is organized into progressive stages, each with clear triggers, technical requirements, and migration strategies.

The core principle: **extract and scale incrementally**. Start with the modular monolith from the MVP, identify bottlenecks through real production traffic, and extract services only when the cost of not extracting exceeds the cost of extraction.

---

## 2. Scaling Stages Overview

| Stage | Triggers | Key Changes | Timeline |
|-------|----------|-------------|----------|
| Stage 1: MVP Growth | 500+ orders/day, 200+ vendors | Vertical scaling, Elasticsearch, basic observability | Months 5-8 |
| Stage 2: Service Extraction | 2,000+ orders/day, deployment bottlenecks | Extract critical services (Order, Dispatch), add Kafka | Months 9-14 |
| Stage 3: Multi-Country | 2+ country launches | Per-country deployments, federation layer | Months 15-20 |
| Stage 4: Feature Expansion | Pro, Wallet, BNPL demand | New service modules, fintech integrations | Months 21-30 |
| Stage 5: Full Microservices | 20,000+ orders/day, 5+ teams | Complete service extraction, Kubernetes, service mesh | Months 31+ |

---

## 3. Stage 1: MVP Growth (500+ Orders/Day)

### 3.1 Triggers

- Daily orders exceed 500 consistently
- PostgreSQL search latency exceeds 500ms at p95
- Backend CPU utilization sustained above 60%
- Team expands beyond 5 engineers

### 3.2 Infrastructure Changes

#### Task S1-INFRA-01: Add Elasticsearch for Search
**Description**: Deploy Elasticsearch cluster (3 nodes) for vendor and item search. Migrate search queries from PostgreSQL full-text to Elasticsearch. Configure Arabic and English analyzers per `docs/16_search_discovery_system.md`.
**Dependencies**: MVP Search Module
**Acceptance Criteria**:
- Elasticsearch cluster running with 3 data nodes
- Vendor index created with mappings for: name, name_ar, cuisine_types, location (geo_point), menu_items (nested)
- Arabic analyzer configured: diacritics removal, letter normalization, transliteration
- Search latency reduced to < 200ms at p95
- Data sync: PostgreSQL → Elasticsearch via change data capture (Debezium) or periodic batch
- Fallback to PostgreSQL search if Elasticsearch unavailable

#### Task S1-INFRA-02: Vertical Scaling of Database & Backend
**Description**: Upgrade PostgreSQL to a larger instance class. Add read replica for reporting and listing queries. Scale backend to 3+ instances behind load balancer.
**Dependencies**: None
**Acceptance Criteria**:
- PostgreSQL primary: upgrade to db.r5.large (2 vCPU, 16GB RAM)
- 1 read replica added for vendor listing, menu reads, order history queries
- Backend scaled to 3 instances (from 2) with auto-scaling policy
- Connection pooling via PgBouncer (200 max connections per replica)
- Database CPU utilization sustained below 50%

#### Task S1-INFRA-03: Enhanced Monitoring & Alerting
**Description**: Implement comprehensive monitoring: APM (New Relic or Datadog), infrastructure metrics (Prometheus + Grafana), error tracking (Sentry), and custom business metrics dashboard.
**Dependencies**: None
**Acceptance Criteria**:
- APM: request latency, error rate, throughput per endpoint
- Infrastructure: CPU, memory, disk, network per service
- Database: query latency, connection pool, replication lag
- Business metrics: orders/hour, average delivery time, rider utilization
- Alerting: PagerDuty integration for critical alerts
- SLA monitoring: 99.5% uptime dashboard

### 3.3 Estimated Cost Increase

| Component | MVP Cost | Stage 1 Cost | Increase |
|-----------|---------|-------------|----------|
| Backend compute | $200 | $400 | +$200 |
| PostgreSQL | $150 | $400 | +$250 |
| Elasticsearch | $0 | $200 | +$200 |
| Monitoring | $0 | $100 | +$100 |
| **Monthly Total** | **$400-640** | **$1,100-1,500** | **+$700-860** |

---

## 4. Stage 2: Service Extraction (2,000+ Orders/Day)

### 4.1 Triggers

- Deployment of one module requires full system restart
- Order module needs independent scaling (10x traffic during peaks)
- Dispatch module requires specialized compute (real-time scoring)
- Team grows to 3+ sub-teams needing independent deployment cadence

### 4.2 Service Extraction Strategy

The extraction follows the **Strangler Fig Pattern**: extract one service at a time, starting with the most bottlenecked module, while maintaining the monolith as the coordination layer.

```
Monolith (MVP)
        │
        ├── Extract Order Service ──→ First microservice
        │   (most traffic, independent scaling need)
        │
        ├── Extract Dispatch Service ──→ Second microservice
        │   (real-time requirements, specialized compute)
        │
        ├── Extract Payment Service ──→ Third microservice
        │   (PCI compliance isolation, security boundary)
        │
        └── Remaining modules stay in monolith
            (User, Vendor, Cart, Search, Notification, Voucher)
```

### 4.3 Tasks

#### Task S2-EXTRACT-01: Add Apache Kafka as Event Bus
**Description**: Deploy Kafka cluster (3 brokers) to replace the in-process event bus. This is a prerequisite for service extraction, as inter-service communication requires durable, async messaging.
**Dependencies**: None (can be done in parallel with Stage 1)
**Acceptance Criteria**:
- Kafka cluster running with 3 brokers
- Topics created: `order.events`, `payment.events`, `dispatch.events`, `inventory.events`, `notification.events`
- Events follow CloudEvents spec (matching `docs/09_backend_architecture.md` Section 4.3)
- Producer/consumer clients configured in existing monolith
- Consumer lag monitoring via Kafka Manager
- At-least-once delivery guarantee

#### Task S2-EXTRACT-02: Extract Order Service
**Description**: Extract the Order module from the monolith into an independent service with its own database schema. Communication with other modules via Kafka events and REST/gRPC calls.
**Dependencies**: S2-EXTRACT-01
**Acceptance Criteria**:
- Order Service runs as independent process with own database
- Order-related tables migrated to separate schema
- API Gateway routes `/orders/*` to Order Service
- Events published to Kafka: `order.created`, `order.status_changed`, `order.cancelled`
- Events consumed from Kafka: `payment.captured`, `rider.assigned`, `inventory.updated`
- Zero-downtime migration: dual-write period → cutover → decommission old module
- Order placement latency maintained at < 2s

#### Task S2-EXTRACT-03: Extract Dispatch Service
**Description**: Extract the Dispatch module into an independent service. This enables independent scaling for real-time rider assignment and location processing.
**Dependencies**: S2-EXTRACT-01, S2-EXTRACT-02
**Acceptance Criteria**:
- Dispatch Service runs independently with own database (riders, delivery_assignments)
- Consumes `order.created` events from Kafka
- Publishes `rider.assigned`, `rider.location_updated` events
- Rider location processing scales independently
- Firebase RTDB integration maintained
- Dispatch time target: < 3 minutes from order to rider assignment

#### Task S2-EXTRACT-04: Extract Payment Service
**Description**: Extract the Payment module for PCI compliance isolation and independent security audits. Payment data must be in a separate database with restricted access.
**Dependencies**: S2-EXTRACT-01, S2-EXTRACT-02
**Acceptance Criteria**:
- Payment Service runs in isolated network segment
- Payment tables in separate database with encrypted connections
- Consumes `order.created` events for payment initiation
- Publishes `payment.captured`, `payment.failed`, `payment.refunded` events
- Stripe webhook handler isolated from main application
- PCI-DSS compliance: no card data in other services
- Payment success rate maintained at > 98%

### 4.4 Infrastructure at Stage 2

```
┌─────────────────────────────────────────────────┐
│                  API Gateway                      │
└────────┬─────────┬──────────┬─────────┬─────────┘
         │         │          │         │
    ┌────▼───┐ ┌──▼────┐ ┌──▼─────┐ ┌─▼──────────┐
    │ Order  │ │Payment│ │Dispatch│ │ Monolith    │
    │Service │ │Service│ │Service │ │ (User,Vendor│
    └───┬────┘ └───┬───┘ └───┬────┘ │ Cart,Search)│
        │          │         │       └──────────────┘
        └──────────┼─────────┘
                   │
            ┌──────▼──────┐
            │    Kafka    │
            │  (3 brokers)│
            └─────────────┘
```

### 4.5 Estimated Cost at Stage 2

| Component | Monthly Cost |
|-----------|-------------|
| Backend services (6 instances) | $600 |
| PostgreSQL (primary + 2 replicas) | $600 |
| Redis | $100 |
| Elasticsearch (3 nodes) | $300 |
| Kafka (3 brokers) | $300 |
| Firebase RTDB | $200-500 |
| Monitoring | $200 |
| **Monthly Total** | **$2,300-2,600** |

---

## 5. Stage 3: Multi-Country (2+ Countries)

### 5.1 Triggers

- Business decision to launch in a second country
- Data residency requirements (some countries mandate local data storage)
- Currency and language differences require separate configurations

### 5.2 Multi-Country Architecture

Each country operates as an independent deployment with its own database, following the production architecture from `docs/04_database_architecture.md` Section 5.1:

```
Country: UAE (TB_AE)              Country: Egypt (HF_EG)
┌──────────────────────┐          ┌──────────────────────┐
│ API Gateway          │          │ API Gateway          │
│ Order Service        │          │ Order Service        │
│ Dispatch Service     │          │ Dispatch Service     │
│ Monolith (remaining) │          │ Monolith (remaining) │
│ PostgreSQL (UAE)     │          │ PostgreSQL (Egypt)   │
│ Redis (UAE)          │          │ Redis (Egypt)        │
│ Kafka (UAE)          │          │ Kafka (Egypt)        │
└──────────────────────┘          └──────────────────────┘
         │                                  │
         └──────────┬───────────────────────┘
                    │
            ┌───────▼───────┐
            │  Federation   │
            │  Layer        │
            │  (Auth SSO,   │
            │  Analytics,   │
            │  Config)      │
            └───────────────┘
```

### 5.3 Tasks

#### Task S3-MULTI-01: Federation Layer
**Description**: Create a federation layer that provides: SSO authentication across countries, centralized analytics aggregation, global configuration management, and cross-country user account linking.
**Dependencies**: S2-EXTRACT-02
**Acceptance Criteria**:
- User can log in with same account in different countries
- SSO token valid across country-specific API endpoints
- Country switching in app: clears cart, updates API endpoint, preserves auth
- Analytics aggregated across countries in centralized dashboard
- Configuration: feature flags can be set per country

#### Task S3-MULTI-02: Per-Country Database Isolation
**Description**: Set up independent database clusters per country. Migrate country-specific data to local clusters. Implement data residency compliance.
**Dependencies**: None (infrastructure)
**Acceptance Criteria**:
- Each country has own PostgreSQL cluster in local region
- Data residency: UAE data in UAE data center, Egypt data in Egypt data center
- No cross-country database queries (country context in API headers)
- GeoDNS routes API requests to country-specific backend

#### Task S3-MULTI-03: Multi-Currency & Localization
**Description**: Add support for multiple currencies (AED, EGP, SAR, etc.) and per-country localization (different Arabic dialects, country-specific payment methods).
**Dependencies**: S3-MULTI-02
**Acceptance Criteria**:
- Currency formatting based on country context
- Payment methods vary by country (Meeza in Egypt, STC Pay in Saudi, etc.)
- Arabic dialect support: Gulf Arabic, Egyptian Arabic, Iraqi Arabic
- Delivery fee calculations per-country
- Voucher codes scoped to country

---

## 6. Stage 4: Feature Expansion (Pro, Wallet, BNPL)

### 6.1 Triggers

- Revenue from delivery fees alone insufficient for growth
- Customer demand for subscription benefits
- Market demand for digital wallet and BNPL

### 6.2 New Service Modules

| Service | Description | Complexity | Timeline |
|---------|-------------|-----------|----------|
| Subscription Service | Pro membership: free delivery, exclusive offers, family plans | Medium | 8-10 weeks |
| Wallet Service | talabat Pay: balance, top-up, transactions | High | 10-12 weeks |
| BNPL Service | PostPaid: installments, auto-payment, Rewind | Very High | 12-16 weeks |
| Rewards Service | Loyalty points: earn, spend, charity, raffle | Medium | 6-8 weeks |
| DineOut Service | Restaurant reservations: BOGO, capacity booking | Medium | 8-10 weeks |

### 6.3 Key Technical Challenges

| Challenge | Solution |
|-----------|---------|
| Wallet regulatory compliance (KYC/AML) | Partner with licensed financial institution; implement tiered KYC |
| BNPL credit risk assessment | Integrate credit scoring API; start with conservative limits |
| Subscription billing (recurring payments) | Stripe Billing or custom subscription engine |
| Wallet transaction consistency | Two-phase commit for wallet deductions; event sourcing for audit |
| Multi-service payment routing | Payment Service becomes gateway routing to wallet/BNPL/card |

### 6.4 Infrastructure Additions

| Component | Purpose | Monthly Cost |
|-----------|---------|-------------|
| Wallet Service instances | 2 vCPU, 4GB x2 | $150 |
| BNPL Service instances | 2 vCPU, 4GB x2 | $150 |
| Subscription Service instances | 1 vCPU, 2GB x2 | $80 |
| Additional Kafka partitions | Higher throughput | Included |
| **Stage 4 Monthly Add** | | **$380** |

---

## 7. Stage 5: Full Microservices (20,000+ Orders/Day)

### 7.1 Architecture at Full Scale

This matches the production architecture from `docs/09_backend_architecture.md`:

```
┌─────────────────────────────────────────────────┐
│                  API Gateway (Kong)               │
└────────┬──────┬──────┬───────┬──────┬───────────┘
         │      │      │       │      │
    ┌────▼┐ ┌──▼──┐ ┌─▼───┐ ┌▼────┐ ┌▼──────────┐
    │User │ │Vendor│ │Order│ │Pay  │ │Dispatch   │
    │Svc  │ │Svc  │ │Svc  │ │Svc  │ │Svc        │
    └─────┘ └─────┘ └─────┘ └─────┘ └───────────┘
    ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
    │Wallet│ │BNPL  │ │Search│ │Notif │ │Subsc │
    │Svc   │ │Svc   │ │Svc   │ │Svc   │ │Svc   │
    └──────┘ └──────┘ └──────┘ └──────┘ └──────┘
    ┌──────┐ ┌──────┐ ┌──────┐
    │Reward│ │DineOut│ │Chat/ │
    │Svc   │ │Svc   │ │AI Svc│
    └──────┘ └──────┘ └──────┘
```

### 7.2 Kubernetes Migration

| Component | Technology | Configuration |
|-----------|-----------|--------------|
| Container Orchestration | Kubernetes (EKS/GKE) | Multi-cluster per country |
| Service Mesh | Istio | mTLS, traffic management, observability |
| CI/CD | GitHub Actions + ArgoCD | GitOps deployment model |
| Secrets Management | HashiCorp Vault | Dynamic secrets rotation |
| Container Registry | ECR / GCR | Per-country registries |

### 7.3 Key Production Patterns

| Pattern | Implementation | Benefit |
|---------|---------------|---------|
| Circuit Breaker | Istio + custom middleware | Prevent cascade failures |
| Event Sourcing | Kafka + CQRS for Order/Payment | Audit trail, replay capability |
| CQRS | Separate read/write models | Optimize read-heavy queries |
| Saga Pattern | Orchestrated sagas for order flow | Distributed transaction management |
| Feature Flags | Custom Config Service + Firebase Remote Config | Gradual rollout, A/B testing |
| Blue/Green Deployment | ArgoCD + Kubernetes | Zero-downtime deployments |

### 7.4 Estimated Cost at Full Scale (Single Country)

| Component | Monthly Cost |
|-----------|-------------|
| Kubernetes cluster | $1,500-3,000 |
| 15 services (2-5 replicas each) | $2,000-4,000 |
| PostgreSQL (primary + 2 replicas) | $800-1,200 |
| Redis cluster | $300-500 |
| Elasticsearch (5 nodes) | $500-800 |
| Kafka (5 brokers) | $500-800 |
| Firebase RTDB | $500-2,000 |
| CDN + Storage | $200-500 |
| Monitoring stack | $500-1,000 |
| **Monthly Total (1 country)** | **$6,800-13,800** |
| **Per additional country** | **$5,000-10,000** |

---

## 8. Migration Checklist: MVP → Medium-Scale

### 8.1 Pre-Migration Validation

- [ ] MVP handling 500+ orders/day with acceptable performance
- [ ] Monitoring dashboards established with baseline metrics
- [ ] On-call rotation established for production incidents
- [ ] Database backup and recovery tested (RPO < 1 minute, RTO < 30 minutes)
- [ ] Load testing completed: platform handles 5x normal traffic
- [ ] Feature flags infrastructure in place for gradual rollouts

### 8.2 Service Extraction Checklist (Per Service)

- [ ] Identify all consumers of the module's API
- [ ] Define Kafka event contracts (producer/consumer)
- [ ] Set up new service infrastructure (compute, database, CI/CD)
- [ ] Implement dual-write period: write to both monolith and new service
- [ ] Validate data consistency between old and new
- [ ] Cutover: API Gateway routes traffic to new service
- [ ] Decommission old module from monolith
- [ ] Remove dual-write code

### 8.3 Multi-Country Launch Checklist

- [ ] Federation layer deployed and tested
- [ ] SSO authentication working across countries
- [ ] Per-country database provisioned in local region
- [ ] GeoDNS configured for country routing
- [ ] Local payment methods integrated
- [ ] Localization strings completed for target language/dialect
- [ ] Vendor onboarding process ready for new market
- [ ] Local operations team hired or contracted
- [ ] Legal and regulatory compliance verified (data residency, payment licensing)

---

## 9. Technology Evolution Map

| Component | MVP | Stage 1 | Stage 2 | Stage 3-5 |
|-----------|-----|---------|---------|-----------|
| Architecture | Modular monolith | Modular monolith + ES | 3 microservices + monolith | Full microservices |
| Database | PostgreSQL | PostgreSQL + ES | Per-service PostgreSQL | Per-service + read replicas |
| Cache | Redis | Redis | Redis | Redis cluster |
| Search | PostgreSQL FTS | Elasticsearch | Elasticsearch | Elasticsearch (5 nodes) |
| Event bus | In-process | In-process + Kafka | Kafka | Kafka (5 brokers) |
| Real-time | Firebase RTDB | Firebase RTDB | Firebase RTDB | Firebase RTDB → WebSocket |
| Push | FCM | FCM | FCM + HMS | FCM + HMS |
| Payments | Stripe + Cash | Stripe + Cash | + country-specific | + Wallet + BNPL |
| Deployment | Docker Compose | Docker Compose | Kubernetes | Kubernetes + Istio |
| CI/CD | GitHub Actions | GitHub Actions | GitHub Actions + ArgoCD | Full GitOps |
| Monitoring | Basic (Sentry) | New Relic + Prometheus | Full observability | Full observability + custom |

---

## 10. Key Decision Points

| Decision | When to Decide | Options | Default |
|----------|---------------|---------|---------|
| Add Elasticsearch | Search latency > 500ms | PostgreSQL FTS upgrade vs. ES | ES (better long-term) |
| Extract first service | Deployment bottleneck | Order vs. Dispatch vs. Payment | Order (highest traffic) |
| Add Kafka | Need async inter-service comms | Kafka vs. RabbitMQ vs. Redis Streams | Kafka (production-proven at scale) |
| Multi-country | Business mandate | Per-country cluster vs. multi-tenant | Per-country (data isolation) |
| Migrate from Firebase RTDB | RTDB cost > $500/month | Custom WebSocket vs. Socket.io | WebSocket (lowest cost at scale) |
| Add Kubernetes | > 5 services, > 3 teams | Kubernetes vs. ECS vs. Cloud Run | Kubernetes (portable, standard) |
| Launch Wallet/BNPL | Regulatory approval + demand | Build vs. Partner | Partner (faster, compliant) |
