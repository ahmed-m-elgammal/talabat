# 07 — Dispatch & Delivery System

## 1. Overview

Talabat's Dispatch & Delivery System is the logistics backbone that connects vendors (restaurants, dark stores, pharmacies) with customers through a network of riders. The system manages the complete delivery lifecycle — from order dispatching and rider assignment through real-time tracking and delivery confirmation. Operating at scale across 9 MENA countries, the dispatch system must handle heterogeneous fleet compositions (owned fleet, third-party fleet partners, vendor-own-delivery), varying traffic conditions, and diverse geographic layouts from dense urban areas (Dubai, Cairo) to suburban sprawl (Riyadh, Kuwait City).

The system is architected around three core principles: **intelligent dispatching** (matching orders to the optimal rider based on proximity, capacity, and predicted delivery time), **real-time observability** (live GPS tracking, ETA prediction, and proactive delay management), and **flexible fleet management** (supporting both Talabat's own rider fleet and third-party logistics partners).

---

## 2. Fleet Architecture

### 2.1 Fleet Composition

Talabat operates a **hybrid fleet model** combining owned and partnered delivery capabilities:

| Fleet Type | Description | Use Case |
|-----------|-------------|----------|
| **Talabat Owned Fleet** | Full-time salaried riders on Talabat payroll | Core food delivery, TGO guaranteed orders |
| **Third-Party Fleet Partners** | Contracted logistics companies (e.g., local courier services) | Surge capacity, secondary coverage areas |
| **Vendor Own Delivery (VOD)** | Restaurant's own delivery staff | Restaurants with existing delivery capability |
| **Dark Store Pickers** | Warehouse staff who pick and pack grocery orders | Q-Commerce (grocery) orders |

**Vendor Delivery Distinction:**

The app explicitly differentiates between delivery types, as evidenced by translation keys:
- **"Delivered by talabat"** — Talabat's rider handles delivery
- **"Delivered by restaurant"** — Vendor uses own delivery staff
- **Contactless delivery** — Special icon indicating no-contact handoff

### 2.2 Rider Data Model

```
riders
├── id (UUID, PK)
├── first_name, last_name
├── phone_number (E.164 format)
├── country_code
├── vehicle_type (bicycle, motorcycle, car)
├── vehicle_license_plate (nullable)
├── is_active (boolean)
├── current_latitude, current_longitude (real-time updated)
├── current_heading (degrees, 0-360)
├── current_speed (km/h)
├── status (online, offline, on_delivery, on_break)
├── rating_avg (1.0-5.0)
├── total_deliveries
├── fleet_partner_id (nullable — null = owned fleet)
├── max_concurrent_orders (1-3, based on vehicle and area)
├── supported_verticals (food, grocery, pharmacy — array)
├── zones (JSON array of area IDs the rider covers)
└── created_at, updated_at
```

### 2.3 Rider App

Riders use a separate application (not the customer app) with these capabilities:

- **Order acceptance/rejection**: Riders receive push notifications for nearby orders
- **GPS navigation**: Integrated map with turn-by-turn directions to vendor and customer
- **Order lifecycle management**: Mark pickup, in-transit, and delivery status
- **Earnings dashboard**: Daily/weekly earnings, bonus tracking
- **Chat with customer**: In-app messaging for delivery coordination
- **Document management**: License, ID, and vehicle registration upload
- **Cash management**: Track cash collections for cash-on-delivery orders

---

## 3. Dispatching Algorithm

### 3.1 Order Assignment Flow

```
Order Confirmed → Dispatch Engine
        │
        ├── VOD? → Route to Vendor Own Delivery
        │
        ├── Talabat Delivery → Candidate Selection
        │       │
        │       ├── Find riders within radius (geospatial query)
        │       ├── Score candidates (proximity, direction, capacity, rating)
        │       ├── Rank candidates by composite score
        │       │
        │       ├── Auto-assign (if high-confidence match)
        │       │       └── Send assignment push to top candidate
        │       │
        │       └── Broadcast (if no clear match)
        │               └── Send to top N candidates simultaneously
        │                       └── First to accept gets the order
        │
        └── Assignment Confirmed
                ├── Notify customer: "Rider assigned"
                ├── Update Firebase RTDB: delivery_assignments
                └── Start real-time tracking
```

### 3.2 Candidate Scoring

The dispatch engine scores each candidate rider using a weighted composite:

```
score = w1 * proximity_score
      + w2 * heading_score
      + w3 * capacity_score
      + w4 * vertical_match_score
      + w5 * rating_score
      + w6 * predicted_delivery_time_score
```

| Factor | Weight | Description |
|--------|--------|-------------|
| Proximity | 30% | Distance from rider to vendor (closer = better) |
| Heading | 15% | Rider's current direction of travel toward vendor |
| Capacity | 15% | Number of concurrent orders rider can handle |
| Vertical Match | 10% | Rider's supported verticals vs. order vertical |
| Rating | 10% | Rider's average customer rating |
| Predicted Delivery Time | 20% | Estimated total delivery time if this rider is assigned |

### 3.3 Batch Dispatching (Multi-Order)

For Q-Commerce (grocery) orders, the system supports **batch dispatching** where a single rider picks up multiple orders from the same dark store for delivery in the same area:

1. System identifies orders going to the same delivery zone
2. Groups orders by dark store and delivery area
3. Assigns batch to single rider
4. Calculates optimal delivery route (traveling salesman approximation)
5. Rider picks up all orders, then delivers in sequence

**Constraints:**
- Maximum 3 orders per batch
- All orders must be within 2km radius
- Maximum additional delay per order: 10 minutes
- Perishable items (pharmacy, fresh groceries) have priority

### 3.4 Zone-Based Dispatching

The system uses **geofenced delivery zones** managed by the feature flag `ff_ecosystems_geofence_enabled`. Each zone has:

- Defined polygon boundaries (latitude/longitude vertices)
- Associated rider pool
- Vendor coverage mapping
- Demand prediction models per time slot

---

## 4. Real-Time Tracking

### 4.1 GPS Tracking Architecture

```
Rider App (GPS Sensor)
        │
        ▼ (every 5-10 seconds)
Rider Location Service
        │
        ├── Redis: SET rider:{id}:location {lat, lng, heading, speed} EX 10
        ├── Firebase RTDB: SET order_tracking/{order_id}/rider_location
        └── MongoDB: INSERT rider_locations (for analytics)
                │
                ▼ (listener)
Customer App
        │
        ├── Update map marker position
        ├── Recalculate ETA
        └── Update Live Activity notification
```

### 4.2 ETA Prediction

The ETA engine combines multiple data sources for accurate delivery time prediction:

| Input Source | Data | Weight |
|-------------|------|--------|
| Historical delivery data | Average delivery time per area-vendor pair | 30% |
| Real-time traffic | Google Maps / Huawei Maps traffic API | 25% |
| Rider current speed | GPS speed readings | 15% |
| Order complexity | Item count, preparation time | 15% |
| Weather conditions | Weather API integration | 10% |
| Time of day | Peak/off-peak patterns | 5% |

**ETA Updates:**
- Recalculated every 30 seconds during active delivery
- Pushed to customer app via Firebase RTDB
- Displayed as a range: "25-35 min" during preparation, narrowing to "3 min" near delivery

### 4.3 Customer Tracking UI

The customer-facing tracking experience includes:

| Phase | Display | Data Source |
|-------|---------|-------------|
| Order placed | Vendor confirmed, preparing order | Order status |
| Preparing | Estimated preparation time countdown | Vendor POS integration |
| Rider assigned | Rider name, photo, vehicle info | Rider profile data |
| On the way | Live map with rider position, ETA | Firebase RTDB |
| Near delivery | "Almost there" notification, distance countdown | Geofence proximity |
| Delivered | Delivery confirmation, tip prompt | Rider confirmation |

**Map Integration:**
- Google Maps (`google_maps_flutter_android`) for Google Play Services devices
- Huawei Maps (`huawei_map`) for HMS devices
- `map_launcher` plugin for opening in external navigation apps
- Custom map markers for vendor and delivery locations

---

## 5. Delivery Confirmation

### 5.1 Proof of Delivery

| Method | Applicability | Description |
|--------|--------------|-------------|
| **GPS geofence** | Default | Rider enters 50m geofence around delivery address; auto-confirms delivery |
| **Customer PIN** | High-value orders | Customer provides 4-digit PIN from app to rider |
| **Photo proof** | Cash orders, contactless | Rider takes photo of delivered package at doorstep |
| **OTP verification** | Pharmacy orders | Customer enters OTP from SMS to confirm receipt of medication |

### 5.2 Delivery Issues

| Issue | Customer UI | System Response |
|-------|------------|-----------------|
| Rider can't find address | Chat prompt to customer | Rider chat initiated |
| Customer not reachable | Rider calls/WhatsApp | 3-call policy, then mark as "attempted delivery" |
| Wrong order delivered | Help center escalation | Full refund + redelivery |
| Order damaged | Photo upload in help center | Partial/full refund based on damage |
| Delivery too late | Compensation flow | TGO guarantee triggers automatic compensation |

---

## 6. Rider Communication

### 6.1 Rider-Customer Chat

The feature flags `ff_poe_show_flutter_rider_chat` and `ff_poe_show_flutter_rider_tip` enable in-app chat and tipping between riders and customers:

- **In-app messaging**: Text-based chat between rider and customer
- **Predefined messages**: Quick-select options ("I'm at the gate", "Come to lobby", "Leave at door")
- **Image sharing**: Customer can share building entrance photo
- **Chat notifications**: Push notification for new messages via dedicated chat channel

### 6.2 Rider Tips

The rider tipping feature (`ff_poe_show_flutter_rider_tip`) allows customers to add tips:

- Preset amounts: AED 3, 5, 10 (or equivalent per country)
- Custom amount entry
- Tip added to total at checkout or after delivery
- Animation on tip submission (`order.experience.vendor.item.add_animation` style)

---

## 7. Dispatch Analytics & Optimization

### 7.1 Key Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Dispatch time (order to rider assignment) | < 3 min | Time from order confirmation to rider assignment |
| Pickup time (assignment to pickup) | < 15 min | Time from rider assignment to order pickup |
| Delivery time (pickup to delivery) | < 30 min | Time from pickup to customer delivery |
| First-attempt delivery rate | > 95% | Successful delivery on first attempt |
| Rider utilization rate | > 70% | Percentage of online time spent on deliveries |
| Batch efficiency | > 1.5 orders/trip | Average orders per rider trip (Q-Commerce) |

### 7.2 Demand Prediction

The dispatch system uses ML models for demand prediction:

- **Time-series forecasting**: Predict order volume per area per 30-minute window
- **Pre-positioning**: Move riders to high-demand areas before predicted surges
- **Surge detection**: Identify unexpected demand spikes and trigger fleet expansion
- **Weather correlation**: Adjust predictions based on weather conditions

### 7.3 Perseus Analytics Integration

Dispatch-related events tracked through Perseus:

- `rider_assigned` — Order assigned to rider
- `rider_pickup` — Rider picked up order from vendor
- `rider_en_route` — Rider started delivery
- `rider_delivered` — Order delivered to customer
- `rider_chat_initiated` — Customer or rider started chat
- `delivery_delayed` — ETA exceeded by more than 10 minutes
- `dispatch_failed` — No rider available within 10 minutes
