# 04 — Database Architecture

## 1. Overview

Talabat employs a **multi-layered data persistence strategy** that combines local on-device storage (for offline capability, caching, and performance), a remote relational database (for transactional consistency), and real-time document stores (for live order tracking and high-throughput reads). The architecture is designed to support Talabat's multi-vertical platform (Food, Grocery, Pharmacy, DineOut, Flowers) across 9 MENA countries with sub-second response times and offline-first resilience.

The core design philosophy follows the **Offline-First Pattern**: the mobile client caches data locally and synchronizes with the backend when connectivity is restored. This is critical in markets where network reliability varies significantly, particularly in Iraq, Jordan, and parts of Egypt.

---

## 2. Local Storage (On-Device)

### 2.1 SQLite via `sqflite` (Flutter)

The primary local database on the client side is SQLite, accessed through the `sqflite_android` Flutter plugin (`com.tekartik.sqflite.SqflitePlugin`). This provides the foundation for all structured local data storage within the Flutter application layer.

**Key Local Tables (Inferred from Translation Keys and Feature Flags):**

| Table | Purpose | Key Fields |
|-------|---------|------------|
| `cart_items` | Shopping cart persistence across sessions | `item_id`, `vendor_id`, `quantity`, `unit_price`, `selected_options`, `vertical_type` |
| `cached_vendors` | Offline restaurant/store listings | `vendor_id`, `name`, `cuisine_types`, `delivery_time`, `rating`, `logo_url`, `latitude`, `longitude`, `is_open` |
| `cached_menus` | Menu data with TTL-based invalidation | `vendor_id`, `menu_json`, `cached_at`, `ttl_seconds` |
| `saved_addresses` | Delivery address book | `address_id`, `latitude`, `longitude`, `area`, `building`, `floor`, `apartment`, `is_default`, `address_type` |
| `search_history` | Recent search queries | `query`, `vertical`, `searched_at` |
| `order_history` | Past orders for reordering | `order_id`, `vendor_id`, `items_json`, `total`, `ordered_at`, `status` |
| `payment_methods` | Saved payment instruments | `token_id`, `card_last_four`, `card_brand`, `is_default`, `payment_type` |
| `notification_preferences` | Per-channel notification settings | `channel_id`, `is_enabled` |

**Cache Invalidation Strategy:**

The app implements a **TTL-based cache invalidation** system with multiple fallback layers. The feature flags `bae_menu_cache_ttl_config` and `bae_cache_ttl_config` control TTL durations per vertical. The `bae_menu_serve_cache_fallback` flag enables serving stale cache when the API is unreachable, with the UI displaying a toast: `"Couldn't load some information. Try again."` alongside cached vendor listings with the subtitle: `"Looks like you're offline, but here are some picks which you can still explore"`.

### 2.2 Room Database (Perseus Analytics)

The native Android layer uses **Room Database** (`androidx.room`) for the Perseus analytics event queue. This is the only Room database explicitly found in the decompiled code.

**Database:** `TrackingDatabase`

**Table:** `tracking_perseus_events`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `Long` | Primary Key, Auto-increment | Unique event identifier |
| `timestamp` | `Long` | Not Null | Event creation timestamp (epoch ms) |
| `payloadTimeStamp` | `String` | Not Null | Payload generation timestamp |
| `country` | `String` | Not Null | ISO country code (e.g., `AE`, `KW`, `EG`) |
| `advertisingId` | `String?` | Nullable | Google Advertising ID |
| `appId` | `String` | Not Null | Application identifier |
| `appName` | `String` | Not Null | Application display name |
| `appVersionCode` | `String` | Not Null | Version code string |
| `adjustId` | `String?` | Nullable | Adjust attribution ID |
| `userId` | `String` | Not Null | Authenticated user ID |
| `uaId` | `String?` | Nullable | User Acquisition ID |
| `clientId` | `String` | Not Null | Client instance identifier |
| `sessionId` | `String` | Not Null | Current session identifier |
| `sdkVersionName` | `String` | Not Null | Perseus SDK version |
| `globalEntityId` | `String?` | Nullable | Global entity identifier (e.g., `TB_AE`, `HF_EG`) |
| `consent` | `String?` | Nullable | User consent status |
| `sessionOffset` | `Long` | Not Null | Time offset from session start |
| `eventVariables` | `String` | Not Null | JSON blob of event parameters |
| `ecommerceItems` | `String?` | Nullable | JSON array of ecommerce item data |
| `ecommerceComponents` | `String?` | Nullable | JSON array of ecommerce component data |
| `isDebug` | `Boolean` | Not Null | Debug environment flag |
| `eventAction` | `String?` | Nullable | Event action name for logging |
| `appBuildVersion` | `String` | Not Null | Build version string |
| `rechargeTo` | `String?` | Nullable | Recharge target identifier |

**DAO Operations:** `CachedHitEventsDao`

```kotlin
// Pseudocode from decompiled sources
@Dao
interface CachedHitEventsDao {
    @Insert
    suspend fun insertEventTimestamp(event: HitEvent)

    @Query("SELECT * FROM tracking_perseus_events ORDER BY timestamp ASC LIMIT :batchSize")
    suspend fun getHitEvents(batchSize: Int): List<HitEvent>

    @Delete
    suspend fun deleteHitEvents(events: List<HitEvent>)

    @Query("DELETE FROM tracking_perseus_events WHERE timestamp < :timestamp")
    suspend fun deleteEventsOlderThan(timestamp: Long)

    @Query("SELECT MIN(timestamp) FROM tracking_perseus_events")
    suspend fun oldestEventTimestamp(): Long?

    @Query("SELECT COUNT(*) FROM tracking_perseus_events")
    suspend fun sizeOfBacklog(): Int

    @Query("SELECT COUNT(DISTINCT eventAction) FROM tracking_perseus_events")
    suspend fun getNumberOfEventActions(): Int

    @Query("SELECT COUNT(*) as count, MIN(timestamp) as oldest FROM tracking_perseus_events")
    suspend fun loggingInformation(): LoggingInfo
}
```

**Database Migrations (Chronological):**

| Migration | Schema Change |
|-----------|--------------|
| `AddHitsInOneTable` | Initial consolidated table |
| `AddIdToHitEventTimestamp` | Added auto-increment ID column |
| `AddIsDebugMigration` | Added `isDebug` boolean column |
| `AddAppBuildVersionMigration` | Added `appBuildVersion` string column |
| `AddEventActionForLoggingMigration` | Added `eventAction` nullable string column |
| `AlterAnalyticsColumnNotRequiredMigration` | Made analytics columns nullable |
| `AddEnhancedEcommerceObjectsMigration` | Added `ecommerceItems` and `ecommerceComponents` |
| `AddContextualInformationColumnsMigration` | Added contextual data columns |
| `AddRechargeToMigration` | Added `rechargeTo` column |

### 2.3 SharedPreferences

Multiple SharedPreferences stores are used for lightweight key-value persistence:

| Store | Scope | Key Data |
|-------|-------|----------|
| `FlutterSharedPreferences` | Flutter application-wide | Auth tokens, user preferences, feature flags, onboarding state, cart metadata |
| `live_notification_state` | Live Activity service | `active_orders` — JSON array of active order IDs for foreground service |
| `SharedPreferencesLocalStorage` | DeliveryHero persistence lib | Generic cache for API responses, feature configs |
| `UnencryptedSharedPreferencesLocalStorage` | Non-sensitive data | Device info, non-PII settings |

**Critical `FlutterSharedPreferences` Keys (Inferred):**

```
flutter.auth_token              # JWT access token
flutter.refresh_token           # JWT refresh token
flutter.user_id                 # Authenticated user ID
flutter.selected_country       # Current country code
flutter.selected_address_id    # Default delivery address
flutter.onboarding_completed   # Boolean flag
flutter.pro_subscription_status # Pro membership state
flutter.wallet_balance         # talabat Pay balance cache
flutter.feature_flags_cache    # JSON of server-side feature flags
flutter.saved_payment_methods  # JSON of tokenized payment instruments
```

### 2.4 Memory and Disk Cache (DeliveryHero Persistence Library)

The `com.deliveryhero.persistence` package provides a layered caching abstraction:

```
CacheData (interface)
    ├── MemoryCacheImpl    — LRU in-memory cache (HashMap + eviction policy)
    ├── DiskCacheImpl      — File-based disk cache with size limits
    └── DiskCacheLegacyImpl — Legacy disk cache for backward compatibility
```

**Serialization:** Both `GsonSerializer` and `MoshiSerializer` are supported. The `MultiDateFormatGsonSerializer` handles multiple date formats for API compatibility across different backend microservices.

---

## 3. Remote Database Architecture (Server-Side)

### 3.1 Primary Relational Database — PostgreSQL

The backend uses **PostgreSQL** as the primary OLTP database for transactional data requiring ACID compliance. This is inferred from Delivery Hero's published technology stack and the app's transactional requirements.

**Core Schema Domains:**

#### Users & Authentication
```
users
├── id (UUID, PK)
├── first_name, last_name
├── email (unique, nullable)
├── phone_number (unique, E.164 format)
├── country_code
├── date_of_birth
├── gender
├── talabat_reference_number
├── created_at, updated_at
├── is_active
└── consent_flags (JSON)

user_auth_providers
├── id (UUID, PK)
├── user_id (FK → users)
├── provider_type (enum: email, google, apple, facebook)
├── provider_id
├── linked_at

addresses
├── id (UUID, PK)
├── user_id (FK → users)
├── label (home, work, other)
├── latitude, longitude
├── area_name
├── building, floor, apartment
├── delivery_instructions
├── is_default
└── geohash (for area-based vendor matching)
```

#### Vendors (Restaurants, Stores, Pharmacies)
```
vendors
├── id (UUID, PK)
├── name, name_ar
├── legal_name
├── vertical_type (enum: food, grocery, pharmacy, flowers, speciality)
├── country_code
├── area_ids (JSON array)
├── latitude, longitude
├── delivery_radius_km
├── is_open, is_busy
├── delivery_fee_base
├── minimum_order_value
├── estimated_delivery_time_min, estimated_delivery_time_max
├── rating_avg, rating_count
├── cuisine_types (JSON array)
├── brand_id (FK → brands, nullable)
├── tgo_enabled (boolean — Talabat Guaranteed Order)
├── pro_eligible (boolean — free delivery for Pro members)
├── contact_phone, contact_landline
└── created_at, updated_at

vendor_schedules
├── id (UUID, PK)
├── vendor_id (FK → vendors)
├── day_of_week (0-6)
├── open_time, close_time
└── is_closed

vendor_holidays
├── id (UUID, PK)
├── vendor_id (FK → vendors)
├── holiday_date
└── reason
```

#### Menu / Inventory
```
menu_categories
├── id (UUID, PK)
├── vendor_id (FK → vendors)
├── name, name_ar
├── display_order
└── is_available

menu_items
├── id (UUID, PK)
├── category_id (FK → menu_categories)
├── vendor_id (FK → vendors)
├── name, name_ar
├── description, description_ar
├── base_price
├── price_on_selection (boolean)
├── is_available
├── stock_count (nullable — null = unlimited)
├── preparation_time_minutes
├── image_url
├── is_age_restricted (boolean)
├── display_order
└── nutritional_info (JSON, nullable)

item_options
├── id (UUID, PK)
├── item_id (FK → menu_items)
├── name, name_ar
├── is_required
├── min_selections, max_selections
└── display_order

option_choices
├── id (UUID, PK)
├── option_id (FK → item_options)
├── name, name_ar
├── price_modifier
├── is_default
├── is_available
└── calories
```

#### Orders
```
orders
├── id (UUID, PK)
├── order_number (human-readable, indexed)
├── user_id (FK → users)
├── vendor_id (FK → vendors)
├── vertical_type
├── status (enum: ordered, preparing, delivering, delivered, cancelled)
├── delivery_type (enum: delivery, pickup)
├── subtotal
├── delivery_fee
├── service_fee
├── municipality_tax
├── tourist_tax
├── discount_amount
├── voucher_code (nullable)
├── voucher_discount
├── rider_tip
├── total
├── payment_method (enum: card, apple_pay, google_pay, cash, wallet, bnpl, stc_pay, benefit_pay, zaincash)
├── payment_status (enum: pending, paid, failed, refunded)
├── delivery_address_snapshot (JSON)
├── delivery_latitude, delivery_longitude
├── is_tgo (boolean)
├── is_contactless (boolean)
├── scheduled_delivery_time (nullable)
├── actual_delivery_time (nullable)
├── cancellation_reason (nullable)
├── created_at, updated_at
└── placed_at

order_items
├── id (UUID, PK)
├── order_id (FK → orders)
├── menu_item_id (FK → menu_items)
├── item_name, item_name_ar
├── quantity
├── unit_price
├── total_price
├── selected_options (JSON)
├── special_instructions
└── item_status (enum: confirmed, out_of_stock, price_updated, limited_stock, replacement)

order_modifications
├── id (UUID, PK)
├── order_id (FK → orders)
├── modification_type (enum: item_removed, item_added, price_updated, stock_limited)
├── original_total
├── new_total
├── difference_amount
├── created_at
└── reason
```

#### Payments
```
payments
├── id (UUID, PK)
├── order_id (FK → orders)
├── user_id (FK → users)
├── amount
├── currency
├── payment_method_type
├── gateway (enum: checkout_com, hyperpay, wallet, apple_pay, google_pay)
├── gateway_transaction_id
├── gateway_checkout_id
├── token_id (FK → payment_tokens, nullable)
├── status (enum: initiated, authorized, captured, failed, refunded, voided)
├── three_ds_authenticated (boolean)
├── risk_score (nullable)
├── created_at, updated_at

payment_tokens
├── id (UUID, PK)
├── user_id (FK → users)
├── token_type (enum: card, wallet, bnpl)
├── token_value (encrypted)
├── card_last_four
├── card_brand (enum: visa, mastercard, amex, meeza, knet)
├── card_expiry_month, card_expiry_year
├── is_default
├── provider (enum: checkout_com, hyperpay)
└── created_at

bnpl_installments
├── id (UUID, PK)
├── order_id (FK → orders)
├── user_id (FK → users)
├── total_amount
├── payment_due_date
├── payment_status (enum: unpaid, paid, overdue, cancelled, failed, processing, refunded, paused)
├── auto_payment_method_id (FK → payment_tokens, nullable)
├── preferred_payment_day (nullable)
├── is_auto_payment_enabled (boolean)
└── created_at, updated_at
```

#### Subscriptions (talabat Pro)
```
subscriptions
├── id (UUID, PK)
├── user_id (FK → users)
├── plan_type (enum: pro, pro_lite)
├── status (enum: active, cancelled, expired, trial)
├── trial_start_date (nullable)
├── trial_end_date (nullable)
├── subscription_start_date
├── subscription_end_date
├── auto_renew (boolean)
├── auto_upgrade_enabled (boolean)
├── payment_method_id (FK → payment_tokens)
├── monthly_fee
├── currency
├── savings_to_date
├── family_plan_members (JSON array of phone numbers)
├── cancelled_at (nullable)
├── cancellation_reason (nullable)
└── created_at, updated_at
```

#### Dispatch / Delivery
```
delivery_assignments
├── id (UUID, PK)
├── order_id (FK → orders)
├── rider_id (FK → riders)
├── status (enum: assigned, picked_up, on_the_way, delivered)
├── assigned_at
├── picked_up_at
├── delivered_at
├── rider_latitude, rider_longitude (real-time updated)
├── estimated_arrival_minutes
└── delivery_proof_url (nullable)

riders
├── id (UUID, PK)
├── first_name, last_name
├── phone_number
├── vehicle_type (enum: bicycle, motorcycle, car)
├── is_active
├── current_latitude, current_longitude
├── country_code
├── fleet_partner_id (nullable — for third-party fleet partners)
└── rating_avg
```

#### Rewards
```
reward_points
├── id (UUID, PK)
├── user_id (FK → users)
├── points_balance
├── total_earned
├── total_spent
└── updated_at

reward_transactions
├── id (UUID, PK)
├── user_id (FK → users)
├── order_id (FK → orders, nullable)
├── points_amount
├── transaction_type (enum: earn, spend, expire, charity)
├── burn_option (enum: free_delivery, money_off, percentage_off, charity, raffle)
├── created_at
└── expires_at (nullable)

vouchers
├── id (UUID, PK)
├── code (unique, indexed)
├── discount_type (enum: percentage, fixed_amount, free_delivery)
├── discount_value
├── min_order_value
├── max_discount_cap
├── bin_restriction (nullable — card BIN-based targeting)
├── usage_limit_total
├── usage_limit_per_user
├── usage_count
├── valid_from, valid_until
├── is_pro_exclusive (boolean)
├── country_code
└── vertical_type (nullable — null = all verticals)
```

### 3.2 Document Store — MongoDB

Used for high-throughput, schema-flexible data:

| Collection | Purpose | Key Fields |
|------------|---------|------------|
| `order_tracking_events` | Real-time order status timeline | `order_id`, `event_type`, `timestamp`, `metadata` |
| `rider_locations` | Live rider GPS streams | `rider_id`, `lat`, `lng`, `heading`, `speed`, `timestamp` |
| `search_analytics` | Search query logs and click-through | `query`, `results`, `clicked_vendor`, `clicked_position`, `vertical`, `user_id`, `session_id` |
| `feature_flags` | Server-side feature flag configs | `flag_key`, `value`, `targeting_rules`, `country_scope`, `updated_at` |
| `vendor_catalogs` | Denormalized vendor search documents | `vendor_id`, `name`, `cuisines`, `area`, `rating`, `delivery_time`, `search_keywords` |
| `menu_cache` | Denormalized menu data for fast reads | `vendor_id`, `categories_json`, `items_json`, `last_updated`, `hash` |

### 3.3 Redis — Caching & Real-Time

| Redis Data Structure | Purpose | TTL |
|---------------------|---------|-----|
| `vendor:{id}:status` | Real-time open/closed/busy status | 30s |
| `vendor:{id}:menu_hash` | Menu content hash for ETag validation | 5m |
| `cart:{user_id}:{vendor_id}` | Server-side cart state | 24h |
| `session:{user_id}` | Active session data | 30m |
| `ratelimit:otp:{phone}` | OTP request rate limiting | 5m |
| `rider:{id}:location` | Latest rider GPS coordinates | 10s |
| `order:{id}:eta` | Dynamic ETA cache | 30s |
| `search:trending:{country}` | Trending search terms | 15m |
| `feature_flags:all` | Cached feature flag snapshot | 5m |
| `geofence:{area_id}:vendors` | Vendors serving a geofenced area | 10m |

### 3.4 Firebase Realtime Database

Used for live order tracking synchronization between rider app and customer app:

```
order_tracking/{order_id}
├── rider_location
│   ├── latitude
│   ├── longitude
│   ├── heading
│   ├── speed
│   └── updated_at
├── order_status
│   ├── current (preparing | on_the_way | delivered)
│   ├── estimated_arrival_minutes
│   └── updated_at
└── chat_messages
    ├── {message_id}
    │   ├── sender_type (customer | rider | support)
    │   ├── message_text
    │   ├── image_url (nullable)
    │   └── sent_at
```

---

## 4. Data Flow Architecture

### 4.1 Write Path (Order Placement)

```
Customer App → API Gateway → Order Service
                                ├── PostgreSQL: INSERT orders, order_items
                                ├── Redis: SET order:{id}:eta
                                ├── Firebase RTDB: SET order_tracking/{id}
                                ├── Message Queue: PUBLISH order.created
                                └── Perseus: TRACK event (async)
                                        └── Room DB (local) → Batch upload
```

### 4.2 Read Path (Vendor Listing)

```
Customer App → API Gateway → Vendor Service
                                ├── Redis: GET vendor:{area}:list (cache hit?)
                                ├── If miss: PostgreSQL → SELECT vendors WHERE area
                                ├── MongoDB: GET vendor_catalogs (search index)
                                └── Redis: SET vendor:{area}:list (TTL 5m)
```

### 4.3 Real-Time Path (Order Tracking)

```
Rider App → Firebase RTDB: UPDATE rider_location
    → Customer App: LISTEN order_tracking/{order_id}/rider_location
    → LiveNotificationService (Android): UPDATE foreground notification
    → ETA Engine: RECALCULATE estimated arrival
```

---

## 5. Data Consistency & Replication

### 5.1 Multi-Country Database Strategy

Each country operates as a **separate logical tenant** with its own database schema or isolated partition, identified by `global_entity_id` (e.g., `TB_AE` for UAE, `TB_KW` for Kuwait, `HF_EG` for HungerStation Egypt). This ensures:

- **Data sovereignty compliance** (local data residency requirements)
- **Independent scaling** per market
- **Configurable feature rollout** per country via `country_code` scope on feature flags
- **Currency isolation** (AED, BHD, EGP, IQD, JOD, KWD, OMR, QAR, SAR)

### 5.2 Eventual Consistency Model

The system follows an **eventually consistent** model for non-critical data (vendor listings, menus, search indexes) and **strong consistency** for financial transactions (orders, payments, wallet operations). The Perseus analytics pipeline is fully asynchronous with at-least-once delivery guarantees via the Room-based event queue with batch upload.

### 5.3 Cache Coherence

The BAE (Backend-Assisted Evaluation) caching system uses **content hashes** and **ETags** for cache validation. When the `bae_menu_serve_cache_fallback` flag is enabled, stale cache is served when the backend is unreachable, with explicit user notification. Cache TTL is configurable per vertical via `bae_menu_cache_ttl_config` and `bae_cache_ttl_config` feature flags.

---

## 6. Data Security & Compliance

### 6.1 Encryption at Rest

- **PII fields** (email, phone_number, payment tokens) are encrypted at the application layer using AES-256-GCM before database storage
- **Payment token values** use hardware-backed keystore encryption on-device (`token_secure_storage`, `payment_native_storage` plugins)
- **Database-level encryption** for SQLite on Android using SQLCipher-compatible approach via secure storage plugins

### 6.2 Data Residency

- Each country's data is stored in **region-specific infrastructure** (UAE data centers for TB_AE, Kuwait for TB_KW, etc.)
- The `reCAPTCHA Enterprise` integration and `Shield Service` plugin provide bot detection and fraud prevention at the data access layer
- The `Incognia` SDK (App ID: `3df43253-f642-4402-86c0-4333fb93bf73`) provides location-based identity verification without collecting installed app lists

### 6.3 Data Retention

- **Order data**: Retained indefinitely for reordering and dispute resolution
- **Analytics events**: 90-day hot retention, 2-year cold storage
- **Rider location data**: 30-day retention (GPS traces)
- **Search analytics**: 1-year retention for recommendation model training
- **Authentication logs**: 1-year retention per compliance requirements

---

## 7. Database Performance Optimization

### 7.1 Indexing Strategy

| Table | Index | Type | Purpose |
|-------|-------|------|---------|
| `vendors` | `(country_code, area_ids, vertical_type, is_open)` | Composite B-tree | Vendor listing queries |
| `vendors` | `(latitude, longitude)` | GiST (PostGIS) | Geospatial vendor search |
| `orders` | `(user_id, status, created_at DESC)` | Composite B-tree | Order history listing |
| `orders` | `(vendor_id, status, created_at DESC)` | Composite B-tree | Vendor order management |
| `menu_items` | `(vendor_id, is_available, category_id)` | Composite B-tree | Menu filtering |
| `payment_tokens` | `(user_id, is_default)` | Composite B-tree | Default payment method lookup |
| `vouchers` | `(code)` | Unique B-tree | Voucher code validation |
| `vouchers` | `(country_code, valid_from, valid_until)` | Composite B-tree | Voucher eligibility queries |

### 7.2 Read Replicas

- **1 primary + 2 read replicas** per country deployment
- Read replicas serve: vendor listings, menu reads, order history, search queries
- Primary serves: order creation, payment processing, user registration, cart modifications
- **Connection pooling** via PgBouncer with 200 max connections per replica

### 7.3 Sharding Strategy

Orders are sharded by `country_code` at the application level, with each country deployment operating an independent PostgreSQL cluster. Cross-country queries are handled through a federated query layer that aggregates results from multiple clusters.

---

## 8. Backup & Disaster Recovery

| Component | Backup Frequency | Retention | Recovery Point Objective |
|-----------|-----------------|-----------|--------------------------|
| PostgreSQL | Continuous WAL archiving + daily full | 30 days | < 1 minute |
| MongoDB | Daily snapshot + oplog | 14 days | < 1 hour |
| Redis | RDB every 15 min + AOF | 7 days | < 1 minute |
| Firebase RTDB | Daily export to Cloud Storage | 30 days | < 24 hours |
| Room (local) | Not backed up (ephemeral analytics) | N/A | Best effort |
| SQLite (local) | Not backed up (cache data) | N/A | Re-synced from server |

---

## 9. Monitoring & Observability

- **Perseus Analytics**: Custom event tracking with Room-based offline queuing and batch upload to `https://perseus-productanalytics.deliveryhero.net`
- **New Relic**: APM for database query performance, connection pool monitoring
- **Firebase Performance**: Network request tracing for API calls involving database reads
- **Sentry**: Error tracking for database exceptions and constraint violations
- **Delivery Hero Performance Kit**: Screen-level TTI measurement with per-device performance budgets (e.g., TTI_HOME: 726ms high-end, 2000ms low-end)
