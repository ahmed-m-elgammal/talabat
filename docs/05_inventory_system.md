# 05 — Inventory System

## 1. Overview

Talabat's inventory system manages the real-time availability, pricing, and stock levels of items across multiple verticals — Food, Grocery (Q-Commerce), Pharmacy, Flowers, and Speciality Stores. The system must handle fundamentally different inventory models: **infinite stock** for restaurant food items (preparation-based), **finite stock** for grocery and pharmacy items (warehouse-based), and **time-bound availability** for DineOut reservations. This multi-model inventory architecture is a core differentiator that enables Talabat to serve as a unified marketplace across disparate product categories.

The inventory system is tightly integrated with the search and discovery layer (for availability filtering), the order management system (for stock reservation and deduction), and the vendor management portal (for real-time stock updates from restaurants and stores).

---

## 2. Inventory Models by Vertical

### 2.1 Food Vertical — Preparation-Based Inventory

Restaurant food items follow an **infinite stock model** where availability is determined by the vendor's operating status rather than a finite quantity count. A restaurant can prepare unlimited portions of a dish as long as it remains open and the dish is listed on the menu.

**Availability Determinants:**

| Factor | Mechanism | Impact |
|--------|-----------|--------|
| Vendor operating hours | `vendor_schedules` + `vendor_holidays` tables | Entire menu unavailable when closed |
| Item-level availability flag | `menu_items.is_available` (boolean) | Individual item marked unavailable by vendor |
| Category-level availability | `menu_categories.is_available` (boolean) | Entire category hidden (e.g., breakfast after 11 AM) |
| Busy status | `vendors.is_busy` (boolean) | Vendor temporarily not accepting new orders |
| Area-level coverage | `vendors.area_ids` + delivery radius | Vendor hidden for out-of-zone addresses |

**Pre-ordering Mechanism:**

When a vendor is closed but will open later, the app supports **pre-ordering**. The translation key `search.preorder` ("Preorder only") and `order.experience.menu.preorder` indicate that users can place orders in advance. The system calculates the earliest available delivery slot based on:

```
earliest_delivery = vendor_next_open_time + preparation_time + delivery_eta
```

**Menu Channel Control:**

The feature flag `ff_menu_disable_menu_channel` allows server-side control to disable the entire menu channel for a vendor during maintenance or incidents, providing a kill switch at the menu level.

### 2.2 Q-Commerce (Grocery) — Finite Stock Inventory

Grocery and quick-commerce items follow a **finite stock model** with real-time quantity tracking at the dark store / warehouse level. This is the most complex inventory model in the system.

**Stock Tracking:**

| Field | Type | Description |
|-------|------|-------------|
| `stock_count` | Integer (nullable) | Current available quantity; `null` = unlimited |
| `low_stock_threshold` | Integer | Threshold for "few left" badge display |
| `out_of_stock_behavior` | Enum | `hide` / `show_greyed` / `show_with_reminder` |
| `restock_expected_at` | Timestamp (nullable) | Expected restock time for "Remind me" feature |

**Item Replacement Timer:**

The feature flag `ff_qcommerce_item_replacement_timer` enables a time-limited flow where, when an ordered item is found to be out of stock during picking, the customer is given a countdown timer (typically 5 minutes) to select a replacement item. The translation key `qcommerce.item_replacement_timer` confirms this feature. If the timer expires, the item is simply removed from the order and the total is adjusted.

**Verified Pickers / Freshness Guarantee:**

The design system icon `ds_talabat_pickers` and translation key `qcommerce.freshness_guaranteed` indicate a **freshness verification system** where trained pickers select items on behalf of customers. This operates as follows:

1. Customer places grocery order
2. Order is assigned to a "Verified Picker" at the dark store
3. Picker scans and selects items, replacing subpar items with better alternatives
4. If an item is unavailable, the replacement timer is triggered for customer approval
5. Picker confirms order completeness and hands off to rider

**Age-Restricted Items:**

Grocery and pharmacy items can be flagged as age-restricted via `menu_items.is_age_restricted`. The translation key `qcommerce.age_restricted_item_confirmation_title` confirms that a confirmation dialog is shown before adding age-restricted items to cart, and delivery requires age verification upon receipt.

**Single-Use Bag Disclaimer:**

The feature flag `ff_qcommerce_single_use_bag_disclaimer` controls display of environmental messaging about single-use bags, complying with sustainability regulations in certain markets.

### 2.3 Pharmacy — Prescription-Linked Inventory

Pharmacy inventory introduces a **prescription-gated availability model** where certain items require a valid prescription before they can be added to cart and ordered.

**Prescription Flow:**

1. User searches for a medication
2. If prescription-required, a prescription upload prompt appears
3. User uploads prescription image (via `image_picker_android` plugin)
4. Pharmacy reviews prescription (manual process)
5. Upon approval, user receives payment summary: "You'll receive a payment summary after insurance approval"
6. Insurance approval may adjust pricing (co-pay calculation)
7. Order is confirmed and dispatched

**Insurance Integration:**

The pharmacy vertical supports health insurance integration where:

- User selects insurance provider during checkout
- System validates coverage for prescribed items
- Co-pay amount is calculated and displayed
- Insurance approval may take 15-60 minutes
- Payment is only captured after insurance approval

### 2.4 DineOut — Capacity-Based Inventory

DineOut reservations use a **capacity-based inventory model** where the stock unit is a **reservation slot** (table + time window) rather than a physical item.

**Availability Model:**

| Parameter | Description |
|-----------|-------------|
| `total_slots` | Maximum reservations per time window |
| `booked_slots` | Currently booked count |
| `available_slots` | `total_slots - booked_slots` |
| `booking_window` | Advance booking horizon (up to 14 days) |
| `min_party_size` / `max_party_size` | Party size constraints per slot |

**Buy 1 Get 1 Packages:**

The DineOut vertical offers **BOGO (Buy 1 Get 1)** packages with separate availability tracking. The translation keys `dine_out.bogo_package` indicate that these packages have limited daily availability and may sell out, requiring real-time decrement on booking.

---

## 3. Inventory Data Flow

### 3.1 Stock Update Pipeline

```
Vendor Portal / POS Integration
        │
        ▼
Inventory Service (Backend)
        │
        ├── PostgreSQL: UPDATE menu_items SET stock_count, is_available
        ├── Redis: SET vendor:{id}:item:{item_id}:stock {count} EX 300
        ├── Event Bus: PUBLISH inventory.updated
        │       │
        │       ▼
        │   Search Index: UPDATE vendor_catalogs (availability facet)
        │       │
        │       ▼
        │   Customer App: WebSocket/SSE push or next API poll refresh
        │
        └── MongoDB: INSERT inventory_audit_log
```

### 3.2 Stock Reservation Flow (Order Placement)

```
Customer places order
        │
        ▼
Order Service → Inventory Service: RESERVE items
        │
        ├── Check stock availability (Redis → PostgreSQL fallback)
        ├── For finite stock items: DECREMENT stock_count atomically
        ├── For infinite stock items: CHECK is_available flag only
        ├── Create stock reservation with 15-minute TTL
        │
        ├── Success → Order proceeds to payment
        └── Failure → Return out_of_stock items, trigger replacement flow
```

### 3.3 Stock Deduction vs. Reservation

The system uses a **two-phase inventory model**:

1. **Reservation Phase**: When an item is added to cart, a soft reservation is created with a short TTL (typically 10-15 minutes). This prevents overselling without permanently committing stock.
2. **Deduction Phase**: When the order is confirmed and payment is captured, the reservation is converted to a permanent deduction. If the reservation expires (cart abandonment), stock is released back.

```
Cart Add → RESERVE(item_id, qty, ttl=10min)
Order Confirm → DEDUCT(item_id, qty) [converts reservation]
Cart Expire → RELEASE(item_id, qty) [reverts reservation]
Order Cancel → RESTOCK(item_id, qty) [returns to available]
```

---

## 4. Menu Architecture

### 4.1 Menu Data Structure

Menus are organized in a **three-level hierarchy**: Vendor → Category → Item → Options → Choices.

```
Vendor (Restaurant/Store)
├── Category: "Starters"
│   ├── Item: "Hummus"
│   │   ├── Option: "Size" (Required, 1 selection)
│   │   │   ├── Choice: "Regular" (+AED 0)
│   │   │   └── Choice: "Large" (+AED 8)
│   │   └── Option: "Add-ons" (Optional, 0-3 selections)
│   │       ├── Choice: "Extra Olive Oil" (+AED 2)
│   │       ├── Choice: "Pita Bread" (+AED 3)
│   │       └── Choice: "Garlic" (+AED 1)
│   └── Item: "Falafel Plate"
│       └── (no options — fixed price)
├── Category: "Main Course"
│   └── ...
└── Category: "Beverages"
    └── ...
```

### 4.2 Menu Caching (BAE System)

The **BAE (Backend-Assisted Evaluation)** caching system optimizes menu loading performance:

| Feature Flag | Purpose |
|-------------|---------|
| `bae_menu_serve_cache_fallback` | Serve stale menu cache when API unreachable |
| `bae_menu_cache_ttl_config` | Configure TTL for menu cache per vertical |
| `bae_cache_ttl_config` | General cache TTL configuration |
| `ff_menu_delete_persistent_cache` | Kill switch to force cache invalidation |
| `ff_menu_show_favorites` | Show favorite items within menu |

**Menu Cache Flow:**

1. App requests menu from API with `If-None-Match: {etag}` header
2. If menu unchanged (304 Not Modified), use local cache
3. If menu changed, API returns full menu with new ETag
4. App stores menu JSON in SQLite with ETag hash
5. Feature flag `exp_search_food_results_cache` and `exp_search_food_results_cache_new_key` control caching at the search/results level

### 4.3 Menu Item Display Logic

The app applies complex display rules based on inventory status and feature flags:

| Condition | Display Behavior |
|-----------|-----------------|
| Item available, in stock | Normal display, "Add" button active |
| Item available, low stock | Normal display + "Few left" badge |
| Item out of stock, `out_of_stock_behavior=hide` | Item completely hidden from menu |
| Item out of stock, `out_of_stock_behavior=show_greyed` | Item shown greyed with "Out of stock" label |
| Item out of stock, `out_of_stock_behavior=show_with_reminder` | Item shown with "Remind me" button |
| Item `price_on_selection=true` | Price shows as "Price on selection" instead of amount |
| Item `is_age_restricted=true` | Age confirmation dialog before add |
| Item has `limited_stock` in order modification | "Limited stock" badge shown in order details |
| Vendor is busy | "Busy" badge shown, item still orderable |

---

## 5. Inventory Synchronization

### 5.1 Real-Time Sync Mechanisms

| Mechanism | Direction | Latency | Use Case |
|-----------|-----------|---------|----------|
| Firebase Realtime Database | Server → Client | < 1s | Live stock updates for active vendor pages |
| SSE (Server-Sent Events) | Server → Client | < 2s | Menu availability changes during browsing |
| API Polling | Client → Server | 30-60s | Fallback when push is unavailable |
| Vendor Portal WebSocket | Vendor → Server | < 1s | Real-time stock updates from restaurant POS |
| POS Integration (API) | POS → Server | < 5s | Automated stock updates from partner POS systems |

### 5.2 Conflict Resolution

When stock data conflicts arise (e.g., customer adds item to cart but stock was just depleted by another order), the system follows this resolution strategy:

1. **Optimistic locking**: Cart add operations use `stock_count > 0` as a conditional check
2. **On conflict**: If the item is no longer available, the cart add fails with the error message: `"Issue adding item to cart, try again"`
3. **Partial fulfillment**: For grocery orders, if some items are unavailable during picking, the item replacement timer is triggered rather than cancelling the entire order
4. **Order modification**: If items become unavailable after order placement, the `order_modifications` table records the change, and the customer is notified with: `"order.details.out_of_stock"` and `"order.details.price_updated"`

### 5.3 Batch Inventory Updates

Vendors can update inventory in bulk through:

1. **Vendor Portal**: Web-based management interface for manual updates
2. **POS Integration**: Automated two-way sync with popular POS systems
3. **API Batch Endpoint**: RESTful endpoint for bulk stock updates (used by large chains)
4. **Scheduled Imports**: CSV/Excel import for vendors without POS integration

---

## 6. Inventory Feature Flags & Experiments

| Flag | Type | Description |
|------|------|-------------|
| `ff_qcommerce_item_replacement_timer` | Feature | Enable timed replacement flow for out-of-stock grocery items |
| `ff_qcommerce_pdp_stepper_migration` | Migration | New product details page with stepper UI for quantity selection |
| `ff_qcommerce_plp_filters_fe` | Feature | Frontend-driven filters on product listing pages |
| `ff_qcommerce_vlp_bae_caching` | Feature | BAE caching for vendor landing pages |
| `ff_qcommerce_dialects` | Localization | Dialect-specific product names and descriptions |
| `ff_qcommerce_single_use_bag_disclaimer` | Compliance | Show environmental disclaimer for single-use bags |
| `exp_qcommerce_show_dynamic_ads` | Experiment | Show sponsored/dynamic ads in product listings |
| `exp_qcommerce_lists_experience` | Experiment | Shopping list feature for multi-item search |
| `exp_qcommerce_photo_to_list` | Experiment | Photo-to-list feature (take photo of items to add to cart) |
| `exp_qcommerce_multi_search` | Experiment | Search multiple items simultaneously |
| `ff_menu_show_favorites` | Feature | Show favorite items in menu |
| `ff_menu_disable_menu_channel` | Kill Switch | Disable entire menu for vendor |
| `ff_menu_delete_persistent_cache` | Kill Switch | Force delete persistent menu cache |
| `ff_menu_item_details_micro_interactions_enabled` | Feature | Micro-interactions on item detail page |
| `ff_ordering_cart_cache_enabled` | Feature | Server-side cart caching |
| `ff_ordering_clear_cart_on_404` | Feature | Clear cart when vendor returns 404 |

---

## 7. Inventory Analytics & Monitoring

### 7.1 Stock-Level Metrics

| Metric | Source | Purpose |
|--------|--------|---------|
| Stock-out rate per vendor | Inventory audit log | Vendor performance monitoring |
| Time-to-restock | `restock_expected_at` tracking | Supply chain optimization |
| Cart abandonment due to stock-out | Perseus event tracking | Revenue impact analysis |
| Replacement acceptance rate | Order modifications data | Customer satisfaction |
| Picking accuracy | Verified Picker scan logs | Quality control |

### 7.2 Perseus Event Tracking

Inventory-related events tracked through the Perseus analytics pipeline:

- `item_viewed` — Item detail page viewed
- `item_added_to_cart` — Item added to cart (with stock status)
- `item_out_of_cart_stock` — Stock-out during cart add attempt
- `item_replacement_offered` — Replacement suggestion shown
- `item_replacement_accepted` — Customer accepted replacement
- `item_replacement_declined` — Customer declined replacement
- `item_reminder_set` — Customer set "Remind me" for out-of-stock item
- `menu_cache_hit` / `menu_cache_miss` — Menu caching effectiveness

---

## 8. Scalability Considerations

### 8.1 Peak Load Handling

During peak hours (lunch 11 AM–2 PM, dinner 7 PM–10 PM), inventory updates can spike to 10,000+ per minute per country. The system handles this through:

- **Redis-based hot path**: Stock checks hit Redis first (sub-millisecond latency)
- **PostgreSQL async writes**: Stock updates are written to PostgreSQL asynchronously via event queue
- **Batch aggregation**: Multiple stock updates for the same vendor are batched before database write
- **Read replicas**: Menu reads served from read replicas to reduce primary load

### 8.2 Dark Store (Q-Commerce) Scale

Each dark store carries approximately 2,000–5,000 SKUs. With 50+ dark stores per major city and 9 countries, the total SKU count exceeds 2 million items. The inventory system must process:

- ~500,000 stock updates per hour across all dark stores
- ~200,000 concurrent cart operations during peak
- < 100ms stock check latency for cart add operations
