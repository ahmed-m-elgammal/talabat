# 06 — Order Management System

## 1. Overview

Talabat's Order Management System (OMS) orchestrates the complete lifecycle of orders from initial cart creation through final delivery confirmation. The system handles orders across five verticals (Food, Grocery, Pharmacy, DineOut, Flowers) with distinct fulfillment workflows, while maintaining a unified order state machine and consistent customer experience. The OMS processes millions of orders daily across 9 MENA countries, requiring high throughput, strong consistency for financial operations, and real-time status updates for both customers and riders.

The OMS is the central nervous system of the platform, integrating with the Inventory System (stock reservation and deduction), Payment System (authorization and capture), Dispatch System (rider assignment and tracking), and Notification System (status change alerts). It must handle complex scenarios including order modifications after placement, partial fulfillment for grocery orders, prescription-based pharmacy orders, and multi-item cancellations with differential refund logic.

---

## 2. Order State Machine

### 2.1 Core Order States

```
                    ┌──────────────────┐
                    │                  │
                    ▼                  │
  ┌─────────┐  ┌─────────┐  ┌─────────┤  ┌───────────┐
  │ ORDERED │─▶│PREPARING│─▶│DELIVERING│─▶│ DELIVERED │
  └────┬────┘  └────┬────┘  └─────────┘  └───────────┘
       │            │
       │            │
       ▼            ▼
  ┌──────────┐  ┌──────────┐
  │CANCELLED │  │CANCELLED │
  └──────────┘  └──────────┘
```

**State Definitions:**

| State | Translation Key | Description | Trigger |
|-------|----------------|-------------|---------|
| `ORDERED` | `order.details.ordered` | Order placed, payment authorized, awaiting vendor acceptance | Payment authorized |
| `PREPARING` | `order.details.preparing` | Vendor confirmed and is preparing the order | Vendor accepts order |
| `DELIVERING` | `order.details.delivering` | Rider has picked up order and is en route | Rider confirms pickup |
| `DELIVERED` | `order.details.delivered` | Order delivered to customer | Rider confirms delivery or customer confirms receipt |
| `CANCELLED` | `order.details.cancelled` | Order cancelled (by customer, vendor, or system) | Cancellation event |

### 2.2 Sub-States and Extended Status

For the active order card displayed on the home screen, additional sub-states provide granular tracking:

| Sub-State | Display Context | Description |
|-----------|----------------|-------------|
| `ordered` | Home card | Order placed, waiting for vendor confirmation |
| `preparing` | Home card | Vendor is preparing the order |
| `delivering` | Home card | Rider is on the way |
| `delivered` | Home card | Order completed |
| `cancelled` | Home card | Order was cancelled |
| `replacement_needed` | Home card | Grocery item needs customer approval for replacement |

### 2.3 Pickup Order Flow

For pickup orders, the state machine is simplified:

```
ORDERED → PREPARING → READY_FOR_PICKUP → PICKED_UP
```

The translation key `search.pickup_blocking_model_header` ("This is a Pickup order for you to self-collect at the restaurant") and `search.pickup_blocking_model_confirm` ("I'll pick it up") confirm that pickup orders display a confirmation modal requiring the customer to acknowledge self-collection responsibility.

### 2.4 DineOut Order Flow

DineOut reservations follow a distinct state machine:

```
RESERVED → CONFIRMED → DINING → COMPLETED
                │
                ▼
            CANCELLED
```

With cancellation policies enforced: `"cancellation up to 24h before"` the reservation time.

---

## 3. Cart Management

### 3.1 Cart Architecture

The cart system supports **single-vendor carts** — a customer can only have items from one vendor in their cart at a time. Adding items from a different vendor triggers a cart clear confirmation dialog.

**Cart Data Model:**

```json
{
  "cart_id": "uuid",
  "user_id": "uuid",
  "vendor_id": "uuid",
  "vertical_type": "food|grocery|pharmacy|flowers",
  "items": [
    {
      "menu_item_id": "uuid",
      "quantity": 2,
      "unit_price": 25.00,
      "selected_options": [
        {
          "option_id": "uuid",
          "choice_ids": ["uuid1", "uuid2"]
        }
      ],
      "special_instructions": "No onions",
      "total_price": 58.00
    }
  ],
  "subtotal": 116.00,
  "delivery_fee": 5.00,
  "service_fee": 2.50,
  "discount": 0.00,
  "total": 123.50,
  "created_at": "2026-01-15T10:30:00Z",
  "expires_at": "2026-01-15T10:45:00Z"
}
```

### 3.2 Cart Operations

| Operation | API | Client Behavior |
|-----------|-----|-----------------|
| Add item | `POST /cart/items` | Accessibility: "Adding to cart" → "Item successfully added to cart" or "Issue adding item to cart, try again" |
| Update quantity | `PUT /cart/items/{id}` | Stepper UI with +/- controls (PDP stepper migration via `ff_qcommerce_pdp_stepper_migration`) |
| Remove item | `DELETE /cart/items/{id}` | Swipe-to-delete with confirmation |
| Clear cart | `DELETE /cart` | Triggered when switching vendors; confirmation dialog: "Clear cart?" |
| Apply voucher | `POST /cart/voucher` | BIN-based voucher validation with error handling |
| Get suggested items | `GET /cart/suggested` | Accessibility: "Item {itemNumber} of {totalItemsCount}" |

### 3.3 Cross-Vendor Cart Handling

When a customer with items from Vendor A in their cart tries to add items from Vendor B:

1. App detects different `vendor_id`
2. Bottom sheet displays: `"address_list.clear_cart.title"` / `"address_list.clear_cart.description"`
3. Customer can choose:
   - **"New order"** — Clear cart and start fresh with Vendor B
   - **Cancel** — Return to Vendor A's menu
4. Feature flag `ff_ordering_clear_cart_on_404` automatically clears cart if Vendor A is no longer available (404 response)

### 3.4 Cart Caching

The feature flag `ff_ordering_cart_cache_enabled` enables server-side cart caching in Redis with key `cart:{user_id}:{vendor_id}` and a 24-hour TTL. This ensures:

- Cart persists across app restarts
- Cart is synchronized across devices (when user logs in on a new device)
- Cart can be recovered after app crashes

---

## 4. Checkout Flow

### 4.1 Checkout Steps

```
Cart Review → Address Selection → Time Selection → Payment → Order Confirmation
```

**Step 1: Cart Review**
- Verify all items are still available and prices haven't changed
- Display basket listing with item count formatting (i18n plural rules: `basket_listing.numItems1`, `numItems2`, `numItems3to10`, `numItems11Plus`)
- Show disclaimer: `basket_listing.disclaimer`
- Apply any valid vouchers or promotions

**Step 2: Address Selection**
- Default address pre-selected
- Option to change: `"accessibility.checkout.address.pre"` ("Delivering to") + `"accessibility.checkout.address.button"` ("Change address")
- Address validation: Check if vendor delivers to selected area
- Area split popup when delivery zone changes: `"address_list.area_split_popup.title"`

**Step 3: Time Selection**
- **ASAP delivery** — Default option, shows estimated delivery time
- **Scheduled delivery** — `search.scheduled_delivery` ("Schedule for later"), available when vendor supports pre-orders
- Pickup time slot for pickup orders

**Step 4: Payment**
- Payment method selection (see Section 5)
- Voucher/BIN voucher application
- Order total breakdown display
- Pro discount application (free delivery if vendor is Pro-eligible)

**Step 5: Order Confirmation**
- Final order summary display
- Payment authorization
- Order number generation
- Real-time tracking activation

### 4.2 Checkout Feature Flags

| Flag | Impact |
|------|--------|
| `exp_ordering_call_checkout_v2` | New checkout v2 API with improved error handling |
| `exp_ordering_incentives_footer` | Show incentive offers in checkout footer |
| `exp_ordering_quick_add` | Quick add items during checkout |
| `exp_ordering_offer_tag` | Display offer tags on checkout items |
| `ff_checkout_enhance_stti` | Enhanced STTI (Service Time To Interactive) performance |
| `ff_pay_button_charge_warning` | Show charge warning on pay button |
| `ff_show_full_cta_on_cart` | Show full CTA text on cart button |
| `ff_show_price_on_checkout_cta` | Show price on checkout CTA button |

### 4.3 Offline Checkout Handling

When the device goes offline during checkout:

- Display message: `"checkout.offline.message"`
- Show refresh option: `"checkout.offline.experience.tap.to.refresh"`
- Cart data is preserved locally
- Payment authorization is deferred until connectivity is restored

---

## 5. Order Pricing & Fee Calculation

### 5.1 Price Components

| Component | Translation Key | Calculation |
|-----------|----------------|-------------|
| Subtotal | `order.details.subtotal` | Sum of (item_price × quantity) for all items |
| Delivery Fee | `order.details.deliveryFee` | Base fee + distance surcharge; `order.details.freeDeliveryFee` for Pro |
| Service Fee | `order.details.serviceFee` | Percentage of subtotal (typically 5-10%) |
| Municipality Tax | `order.details.munciplicity.tax` | Government-imposed tax on food services |
| Tourist Tax | `order.details.tourist_tax` | Applicable in tourist zones (e.g., Dubai) |
| Discount | `order.details.discount` | Applied promotion amount |
| Voucher Discount | `order.details.discount.voucher` | Voucher-specific discount |
| Rider Tip | `order.details.rider.tip` | Customer-specified tip amount |
| Free Item | `order.details.free.item` | Complimentary item from promotion |
| Total | `order.details.total` | Sum of all components |

### 5.2 Dynamic Pricing

- **Surge pricing**: Delivery fees may increase during peak hours or bad weather
- **Distance-based delivery fee**: Calculated from vendor to delivery address
- **Minimum order value (MOV)**: Per-vendor MOV enforced; Pro members may have reduced MOV: `"subscription.mov.title.minimum"` / `"subscription.mov.title.freeDelivery"` / `"subscription.mov.value"` (`{currency} {MOV}`)
- **Pro free delivery**: Pro members receive free delivery on Pro-eligible vendors

---

## 6. Order Modifications

### 6.1 Post-Placement Modifications

Orders can be modified after placement in specific scenarios:

| Scenario | Customer Experience | System Behavior |
|----------|-------------------|-----------------|
| Item out of stock | `"order.details.out_of_stock"` notification | Item removed from order, total adjusted |
| Item price changed | `"order.details.price_updated"` notification | New price applied, difference shown |
| Limited stock | `"order.details.limited_stock"` notification | Quantity reduced to available stock |
| Item added by vendor | `"order.details.added.item"` notification | Complimentary item added |
| Free item included | `"order.details.free.item"` notification | Promotional free item included |

### 6.2 Order Modification Data Model

```
order_modifications
├── order_id
├── modification_type (item_removed, item_added, price_updated, stock_limited)
├── original_total
├── new_total
├── difference_amount
├── reason
└── created_at
```

Translation keys for modification display:
- `"order.details.order.changes"` — Header
- `"order.details.order.updated"` — Status badge
- `"order.details.order.updates"` — Updates section title
- `"order.details.original.total"` — Before modification total
- `"order.details.new.total"` — After modification total

### 6.3 BNPL Order Modifications

For Buy Now Pay Later orders, modifications require special handling:

- `"bnpl.order.modification.details.order.changes"` — Changes header
- `"bnpl.order.modification.details.original.total"` — Original total
- `"bnpl.order.modification.details.new.total"` — New total
- `"bnpl.order.modification.details.order.updated"` — Update status
- Installment amounts are recalculated based on new order total
- If total decreases, refund is applied to BNPL balance

---

## 7. Order Cancellation

### 7.1 Cancellation Policy

| Timing | Customer | Vendor | System |
|--------|----------|--------|--------|
| Before vendor accepts | Full refund | N/A | Auto-cancel if vendor doesn't accept within 5 min |
| After acceptance, before preparation | Full refund (vendor discretion) | Can cancel with reason | Cancel if payment fails |
| During preparation | Partial or no refund | Can cancel if items unavailable | Cancel for fraud/compliance |
| After pickup | No refund | N/A | Exception handling only |

### 7.2 Cancellation Reasons

The subscription cancellation survey (`subscription.cancellation.survey`) pattern is also applied to order cancellations, collecting reasons for service improvement:

- Found a better deal elsewhere
- Delivery time too long
- Changed my mind
- Order by mistake
- Other (free text)

### 7.3 Refund Processing

| Payment Method | Refund Mechanism | Processing Time |
|---------------|-----------------|-----------------|
| Credit/Debit Card | Refund to original card | 3-7 business days |
| Apple Pay | Refund to original card | 3-7 business days |
| Google Pay | Refund to original card | 3-7 business days |
| Wallet (talabat Pay) | Wallet credit | Instant |
| BNPL (PostPaid) | BNPL balance credit | Instant |
| Cash | Wallet credit or voucher | 24-48 hours |

---

## 8. Order Tracking

### 8.1 Real-Time Tracking

The order tracking experience is powered by Firebase Realtime Database for live rider location updates:

- **Rider location**: Updated every 5-10 seconds via GPS
- **ETA**: Dynamically recalculated based on rider location, traffic, and remaining distance
- **Status transitions**: Pushed via Firebase RTDB listeners

### 8.2 Live Activity (Android)

The `LiveNotificationService` provides persistent order tracking on Android:

- Runs as a **foreground service** with notification
- Supports Android 14+ foreground service types
- Persists active orders in SharedPreferences (`active_orders` key)
- Auto-expires orders after 2 hours (7,200,000 ms)
- Updates notification with current order status and ETA

### 8.3 TGO (Talabat Guaranteed Order)

The TGO feature provides enhanced tracking guarantees:

- Bottom sheet promoting TGO: "Track your order with constant live updates"
- "On time delivery" promise
- "Chat agents here for you" support
- Dedicated TGO badge and delivery icon (`ds_tgo_badge`, `ds_delivery_tgo`)

### 8.4 Order Tracking States

| State | Customer View | Rider Action |
|-------|--------------|--------------|
| Ordered | "Order placed" | N/A |
| Preparing | "Preparing your order" | Vendor preparing |
| Delivering | "On the way" + live map | Riding to delivery address |
| Near delivery | "Almost there" | Within 500m of delivery address |
| Delivered | "Delivered" | Confirmed delivery |

---

## 9. Reorder Flow

Customers can reorder from their order history:

1. Navigate to order details (`order.details.title`)
2. Tap "Reorder" button
3. System checks item availability at the vendor
4. Available items are added to cart
5. Unavailable items are shown with "no longer available" notice
6. Customer can modify cart before checkout
7. `order.experience.menu.view_basket` provides quick access to cart from menu

---

## 10. Order Analytics & Events

### 10.1 Tracked Events

| Event | Properties | Purpose |
|-------|-----------|---------|
| `order_created` | order_id, vendor_id, total, items_count, vertical, payment_method | Conversion tracking |
| `order_status_changed` | order_id, old_status, new_status, timestamp | Funnel analysis |
| `order_cancelled` | order_id, reason, cancellation_source, time_since_placement | Churn analysis |
| `order_modified` | order_id, modification_type, items_affected, price_difference | Operations monitoring |
| `order_delivered` | order_id, actual_delivery_time, estimated_delivery_time, delivery_delta | SLA monitoring |
| `cart_abandoned` | vendor_id, items_count, cart_value, abandonment_point | Conversion optimization |
| `voucher_applied` | voucher_code, discount_amount, order_total | Promotion effectiveness |
| `rider_tip_given` | order_id, tip_amount, tip_method | Rider satisfaction metrics |

### 10.2 Performance Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Order placement latency | < 2s | Time from "Place Order" tap to confirmation |
| Checkout TTI | 714ms (high-end) / 2000ms (low-end) | Per-device performance budget |
| Order acceptance rate | > 95% | Vendor accepts within 5 minutes |
| Delivery SLA compliance | > 90% | Delivered within estimated time |
| Modification rate | < 10% | Orders with post-placement modifications |
