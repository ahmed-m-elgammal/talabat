# 09 — Backend Architecture

## 1. Overview

Talabat's backend follows a **microservices architecture** deployed on a multi-cloud infrastructure, designed and operated by Delivery Hero's global engineering organization. The system processes millions of daily orders across 9 MENA countries, requiring high availability (99.95%+ uptime), low latency (p99 < 500ms for API responses), and horizontal scalability to handle 10x traffic spikes during peak hours and promotional events (e.g., Ramadan iftar rushes, national day promotions).

The architecture is organized around **domain-driven design (DDD)** principles, with each microservice owning a bounded context and communicating through well-defined APIs and event-driven messaging. The backend is the central integration point for the Flutter mobile client, vendor portal, rider app, and third-party services, exposing RESTful APIs for synchronous operations and using message queues for asynchronous workflows.

---

## 2. High-Level Architecture

### 2.1 System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ Customer │  │  Vendor  │  │  Rider   │  │  Partner APIs    │   │
│  │ App (FL) │  │  Portal  │  │   App    │  │  (POS/Aggregator)│   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │
└───────┼──────────────┼─────────────┼─────────────────┼─────────────┘
        │              │             │                 │
        ▼              ▼             ▼                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        API GATEWAY LAYER                            │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Kong / AWS API Gateway                                      │  │
│  │  ├── Rate Limiting  ├── JWT Validation  ├── Request Routing │  │
│  │  ├── CORS           ├── API Versioning   ├── Circuit Breaker│  │
│  │  └── SSL Termination └── Request Logging  └── Throttling    │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      SERVICE MESH (Istio)                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │  User    │ │  Vendor  │ │  Order   │ │ Payment  │             │
│  │ Service  │ │ Service  │ │ Service  │ │ Service  │             │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │Dispatch  │ │Inventory │ │ Search   │ │Notif.    │             │
│  │Service   │ │Service   │ │ Service  │ │Service   │             │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │Wallet    │ │ BNPL     │ │Analytics │ │Subscribe │             │
│  │Service   │ │ Service  │ │(Perseus) │ │Service   │             │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                          │
│  │Rewards   │ │ DineOut  │ │ Chat/AI  │                          │
│  │Service   │ │ Service  │ │ Service  │                          │
│  └──────────┘ └──────────┘ └──────────┘                          │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       DATA LAYER                                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │PostgreSQL│ │ MongoDB  │ │  Redis   │ │ Firebase │             │
│  │(Primary) │ │(Document)│ │ (Cache)  │ │  RTDB    │             │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                          │
│  │Elastic-  │ │  S3/GCS  │ │  Kafka   │                          │
│  │search    │ │(Objects) │ │(Events)  │                          │
│  └──────────┘ └──────────┘ └──────────┘                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Microservice Catalog

### 3.1 Core Services

| Service | Bounded Context | Primary Database | Key Responsibilities |
|---------|----------------|-----------------|---------------------|
| **User Service** | User identity & profiles | PostgreSQL | Registration, authentication, profile management, consent |
| **Vendor Service** | Restaurant/store management | PostgreSQL + MongoDB | Vendor CRUD, menu management, operating hours, area coverage |
| **Order Service** | Order lifecycle | PostgreSQL | Order creation, status management, modifications, cancellation |
| **Payment Service** | Payment processing | PostgreSQL | Payment authorization, capture, refund, multi-gateway routing |
| **Dispatch Service** | Rider assignment & tracking | PostgreSQL + Redis | Rider matching, assignment, real-time tracking, ETA |
| **Inventory Service** | Stock management | PostgreSQL + Redis | Stock levels, reservation, deduction, menu availability |
| **Search Service** | Discovery & search | Elasticsearch + MongoDB | Full-text search, filtering, sorting, autocomplete |
| **Notification Service** | Push & in-app messages | MongoDB + Redis | FCM/HMS push, Braze integration, channel management |
| **Wallet Service** | talabat Pay wallet | PostgreSQL | Balance, top-up, transactions, transfer |
| **BNPL Service** | PostPaid (Buy Now Pay Later) | PostgreSQL | Installment management, auto-payment, Rewind |
| **Analytics Service (Perseus)** | Event tracking | Room (client) → PostgreSQL | Event collection, batch processing, reporting |
| **Subscription Service** | talabat Pro | PostgreSQL | Plan management, billing, family plans, auto-upgrade |
| **Rewards Service** | Loyalty points | PostgreSQL | Points earn/spend, charity, raffle, vouchers |
| **DineOut Service** | Restaurant reservations | PostgreSQL | Reservation slots, BOGO packages, bill payment |
| **Chat/AI Service** | Customer support & AI | MongoDB | Support chat, ChatGPT integration, rider chat |

### 3.2 Cross-Cutting Services

| Service | Purpose | Technology |
|---------|---------|------------|
| **API Gateway** | Request routing, rate limiting, auth | Kong / Custom |
| **Service Mesh** | Inter-service communication, observability | Istio |
| **Event Bus** | Asynchronous event processing | Apache Kafka |
| **Config Service** | Feature flag management | Custom + Firebase Remote Config |
| **CDN** | Static asset delivery, image optimization | CloudFront / Cloud CDN |
| **Identity Provider** | JWT issuance, OAuth2 flows | Keycloak / Custom |
| **Monitoring** | APM, logging, alerting | New Relic, Sentry, Firebase Performance |

---

## 4. Communication Patterns

### 4.1 Synchronous Communication (REST/gRPC)

| Pattern | Use Case | Protocol |
|---------|----------|----------|
| Client → Backend | API requests from mobile app | HTTPS/REST (JSON) |
| Service → Service (low-latency) | Order → Inventory (stock check) | gRPC (Protobuf) |
| Service → External | Payment gateway integration | HTTPS/REST |

### 4.2 Asynchronous Communication (Event-Driven)

Critical workflows use **Apache Kafka** for event-driven communication:

| Event | Producer | Consumers | Purpose |
|-------|----------|-----------|---------|
| `order.created` | Order Service | Dispatch, Notification, Analytics | Trigger rider assignment, notify customer |
| `order.status_changed` | Order Service | Notification, Analytics, Rewards | Status update notifications |
| `payment.captured` | Payment Service | Order, BNPL, Wallet | Confirm order, update installments |
| `payment.refunded` | Payment Service | Order, Wallet, BNPL | Process refund, update balances |
| `inventory.updated` | Inventory Service | Search, Vendor, Order | Update search index, check pending orders |
| `rider.location_updated` | Dispatch Service | Firebase RTDB, Order | Real-time tracking |
| `user.registered` | User Service | Notification, Rewards, Analytics | Welcome notifications, initial points |
| `subscription.activated` | Subscription Service | Notification, Rewards, Vendor | Pro benefits activation |

### 4.3 Event Schema (CloudEvents)

Events follow the **CloudEvents** specification (as evidenced by the Checkout.com SDK's CloudEvents logging), providing standardized metadata:

```json
{
  "specversion": "1.0",
  "type": "com.talabat.order.created",
  "source": "/services/order",
  "id": "uuid",
  "time": "2026-01-15T10:30:00Z",
  "datacontenttype": "application/json",
  "data": {
    "order_id": "uuid",
    "user_id": "uuid",
    "vendor_id": "uuid",
    "country_code": "AE",
    "vertical_type": "food",
    "total": 85.50,
    "currency": "AED"
  }
}
```

---

## 5. API Gateway

### 5.1 Gateway Configuration

| Feature | Configuration |
|---------|--------------|
| Rate Limiting | 100 req/min per user, 1000 req/min per IP |
| JWT Validation | RS256, public key rotation every 24h |
| CORS | Allowed origins: `*.talabat.*`, `*.deliveryhero.net` |
| SSL Termination | TLS 1.2+, HSTS enabled |
| Request Timeout | 30s default, 60s for search, 120s for file uploads |
| Circuit Breaker | 5 failures in 30s → open for 60s |
| API Versioning | URL path versioning: `/v1/`, `/v2/` |
| Compression | gzip/brotli for responses > 1KB |

### 5.2 Request Routing

```
/api/v1/users/*        → User Service
/api/v1/vendors/*      → Vendor Service
/api/v1/orders/*       → Order Service
/api/v1/payments/*     → Payment Service
/api/v1/dispatch/*     → Dispatch Service
/api/v1/inventory/*    → Inventory Service
/api/v1/search/*       → Search Service
/api/v1/notifications/* → Notification Service
/api/v1/wallet/*       → Wallet Service
/api/v1/bnpl/*         → BNPL Service
/api/v1/subscriptions/* → Subscription Service
/api/v1/rewards/*      → Rewards Service
/api/v1/dineout/*      → DineOut Service
/api/v1/chat/*         → Chat/AI Service
```

---

## 6. Deployment Architecture

### 6.1 Infrastructure

| Component | Technology | Notes |
|-----------|-----------|-------|
| Container Orchestration | Kubernetes (EKS/GKE) | Multi-cluster per country |
| Service Mesh | Istio | mTLS between services |
| Container Registry | ECR / GCR | Per-country registries |
| CI/CD | GitHub Actions + ArgoCD | GitOps deployment model |
| Secrets Management | HashiCorp Vault | Dynamic secrets rotation |
| DNS | Route53 / Cloud DNS | GeoDNS for country routing |

### 6.2 Per-Country Deployment

Each country operates as an independent Kubernetes cluster:

```
Country Deployment (e.g., UAE)
├── Namespace: talabat-production
│   ├── Pod: user-service (3 replicas)
│   ├── Pod: vendor-service (5 replicas)
│   ├── Pod: order-service (5 replicas)
│   ├── Pod: payment-service (3 replicas)
│   ├── Pod: dispatch-service (5 replicas)
│   ├── Pod: search-service (3 replicas)
│   ├── Pod: notification-service (3 replicas)
│   ├── Pod: ... (other services)
│   ├── Pod: api-gateway (3 replicas)
│   └── Pod: kafka-cluster (3 brokers)
├── Namespace: talabat-monitoring
│   ├── Pod: prometheus
│   ├── Pod: grafana
│   └── Pod: alertmanager
└── Namespace: talabat-data
    ├── StatefulSet: postgresql-primary (1) + replicas (2)
    ├── StatefulSet: mongodb (3-node replica set)
    ├── StatefulSet: redis (1 primary + 2 replicas)
    └── StatefulSet: elasticsearch (3 data nodes)
```

### 6.3 Scaling Strategy

| Trigger | Action | Metric |
|---------|--------|--------|
| CPU > 70% | Scale up pods | 5-minute average |
| Memory > 80% | Scale up pods | 5-minute average |
| Request latency p99 > 1s | Scale up pods | 2-minute average |
| Kafka consumer lag > 1000 | Scale up consumer pods | 1-minute check |
| Scheduled scaling | Pre-scale before peak hours | Time-based (11 AM, 7 PM) |
| Event-driven scaling | Scale before promotions | Manual trigger |

---

## 7. Observability Stack

### 7.1 Monitoring & Alerting

| Tool | Purpose | Data Source |
|------|---------|------------|
| New Relic | APM, distributed tracing, error tracking | All microservices |
| Firebase Performance | Mobile client performance | Flutter app |
| Sentry | Error tracking, crash reporting | Flutter app + Backend |
| Delivery Hero Performance Kit | Custom screen metrics | Flutter app |
| Perseus Analytics | Business event tracking | Flutter app → Room → API |
| Prometheus + Grafana | Infrastructure metrics | Kubernetes, databases |
| PagerDuty | On-call alerting | All monitoring sources |

### 7.2 Logging

- **Structured logging** (JSON format) across all services
- **Correlation IDs** propagated through all service calls (X-Correlation-ID header)
- **Log aggregation** via ELK stack (Elasticsearch, Logstash, Kibana)
- **PII masking** in logs (phone numbers, emails, payment data)
- **Log retention**: 30 days hot, 1 year cold storage

### 7.3 Distributed Tracing

The service mesh (Istio) provides automatic distributed tracing:

```
Mobile App → API Gateway → Order Service → Inventory Service
                           → Payment Service → Checkout.com API
                           → Kafka → Dispatch Service → Firebase RTDB
                           → Kafka → Notification Service → FCM
```

Each trace includes:
- Total latency breakdown by service
- Database query times
- External API call times
- Kafka message publish/consume times

---

## 8. Feature Flag Management

### 8.1 Feature Flag Architecture

Feature flags are managed through a combination of **Firebase Remote Config** (for client-side flags) and a custom **Config Service** (for server-side flags). Flags are evaluated per-request based on:

- Country code (e.g., only for `TB_AE`)
- User segment (new vs. existing, Pro vs. non-Pro)
- Device type (high-end vs. low-end)
- Percentage rollout (0-100%)
- A/B experiment group

### 8.2 Flag Categories

| Category | Count | Examples |
|----------|-------|---------|
| Feature Flags (`ff_`) | 40+ | `ff_menu_show_favorites`, `ff_qcommerce_single_use_bag_disclaimer` |
| Experiments (`exp_`) | 15+ | `exp_ordering_call_checkout_v2`, `exp_qcommerce_multi_search` |
| Kill Switches (`ff_killswitch_`, `rc_killswitch_`) | 5+ | `ff_killswitch_incognia`, `rc_killswitch_lazy_widget` |
| BAE Caching (`bae_`) | 4 | `bae_menu_serve_cache_fallback`, `bae_cache_ttl_config` |

### 8.3 Flag Evaluation Flow

```
Client Request (with user context)
        │
        ▼
API Gateway → Config Service
        │
        ├── Load flags from Redis cache (5m TTL)
        ├── If miss, load from PostgreSQL
        │
        ├── Evaluate targeting rules:
        │   ├── Country match?
        │   ├── User segment match?
        │   ├── Percentage bucket?
        │   └── Device class match?
        │
        └── Return evaluated flag values
                │
                ▼
Client caches flags locally (FlutterSharedPreferences)
```

---

## 9. Security Architecture

### 9.1 Network Security

| Layer | Measure |
|-------|---------|
| Perimeter | WAF (AWS WAF / Cloud Armor), DDoS protection |
| Transport | TLS 1.2+ everywhere, mTLS between services |
| API | Rate limiting, input validation, CORS |
| Application | JWT authentication, RBAC, CSRF protection |

### 9.2 Data Security

| Data Type | Protection |
|-----------|-----------|
| PII | AES-256 encryption at rest, TLS in transit |
| Payment data | PCI-DSS Level 1 compliance, tokenization |
| Passwords | bcrypt/Argon2 hashing, never logged |
| API keys | Vault-managed, rotated every 90 days |
| JWT secrets | RS256, key rotation every 24 hours |

### 9.3 Fraud Prevention Pipeline

```
Request arrives
        │
        ├── reCAPTCHA Enterprise: Bot score (0-1)
        ├── Shield Service: Device fingerprint match
        ├── Incognia: Location consistency check
        ├── VM Detection: Emulator check
        │
        ├── Score > threshold → Allow
        ├── Score in gray zone → Challenge (additional verification)
        └── Score < threshold → Block
```

---

## 10. Disaster Recovery

### 10.1 Availability Targets

| Service | SLA | Max Downtime/Month |
|---------|-----|--------------------|
| API Gateway | 99.95% | 21.9 minutes |
| Order Service | 99.95% | 21.9 minutes |
| Payment Service | 99.99% | 4.3 minutes |
| Search Service | 99.9% | 43.8 minutes |
| Notification Service | 99.5% | 3.6 hours |

### 10.2 Failover Strategy

- **Active-Active**: Order, Payment, and Dispatch services run in active-active across availability zones
- **Active-Passive**: Search and Analytics services run in active-passive with 5-minute failover
- **Database failover**: PostgreSQL automatic failover with < 30s switchover via Patroni
- **Redis failover**: Redis Sentinel with automatic failover in < 10s
