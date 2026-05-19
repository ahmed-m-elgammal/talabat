# MVP Overview — Talabat-Like Delivery Marketplace

## Purpose

This folder contains the structured MVP planning package derived from the full Talabat documentation (`docs/04` through `docs/16`). The goal is to define a minimum viable product that can handle **50–100 vendors/items** in a single-country deployment, with a clear path to scale.

## MVP Design Principles

1. **Single-country, single-vertical first**: Launch with food delivery only in one country (e.g., UAE). Additional verticals (grocery, pharmacy) are Phase 2.
2. **Monolith-first backend**: Use a modular monolith rather than full microservices. Service boundaries are defined in code but deployed as one application. Extract to microservices only when scale demands it.
3. **Simple real-time**: Use Firebase Realtime Database for order tracking (as Talabat does). Avoid building a custom WebSocket infrastructure for MVP.
4. **Minimal payment methods**: Support credit/debit card (via a single gateway like Stripe or Checkout.com) plus cash on delivery. Regional wallets and BNPL are Phase 2.
5. **Flutter cross-platform**: Build once for iOS and Android using Flutter, with BLoC/Cubit state management as described in the full architecture.
6. **Progressive auth**: Allow guest browsing; require OTP authentication only for placing orders.

## Scale Target

| Metric | MVP Target | Phase 2 Target |
|--------|-----------|---------------|
| Vendors | 50–100 | 500+ |
| Daily orders | 100–500 | 10,000+ |
| Concurrent users | 50–200 | 5,000+ |
| Countries | 1 | 3+ |
| Verticals | Food only | Food, Grocery, Pharmacy |
| Riders | 20–50 | 500+ |

## Document Index

| File | Scope | Description |
|------|-------|-------------|
| `00_MVP_Overview.md` | This file | Goals, principles, scale targets |
| `01_Backend_MVP_Scope.md` | Backend | Services, APIs, data layer, deployment |
| `02_Frontend_MVP_Scope.md` | Frontend | Flutter app, screens, state management |
| `03_UI_UX_Scope.md` | UI/UX | Design system, screens, interactions |
| `04_Budget_Cost_Efficiency.md` | Budget | Infrastructure costs, team, phasing |
| `05_Realtime_Architecture.md` | Real-time | Order tracking, push notifications, chat |
| `06_Medium_Scale_Roadmap.md` | Phase 2 | Path from MVP to medium-scale |

## How to Read These Plans

Each plan is organized into **tasks**. Every task follows this format:

```
### T[X.Y] — Task Name

**Description:** What this task delivers and why it matters.

**Dependencies:** Which tasks must be completed first.

**Acceptance Criteria:**
- [ ] Measurable, testable condition
- [ ] Another measurable condition

**Phase:** MVP | Phase 2
```

Tasks marked **Phase: MVP** are required for launch. Tasks marked **Phase: Phase 2** are deferred and clearly labeled so they are not accidentally built early.
