# 15 — Notification System

## 1. Overview

Talabat's notification system is a multi-channel, multi-provider infrastructure that delivers timely and relevant messages to customers across push notifications, in-app messages, and transactional alerts. The system integrates **Braze** (formerly Appboy) as the primary engagement platform for marketing and lifecycle campaigns, **Firebase Cloud Messaging (FCM)** and **Huawei Mobile Services (HMS) Push** for transactional push delivery, and custom in-app notification components for real-time order updates.

The notification system serves three distinct objectives: **transactional** (order status, payment confirmations, delivery updates), **engagement** (promotions, offers, re-engagement campaigns), and **operational** (system alerts, feature announcements). Each objective has different priority levels, delivery guarantees, and regulatory requirements, necessitating a sophisticated routing and prioritization framework.

---

## 2. Notification Architecture

### 2.1 System Architecture

```
Backend Services
        │
        ├── Order Service → Kafka: order.status_changed
        ├── Payment Service → Kafka: payment.captured / payment.failed
        ├── Dispatch Service → Kafka: rider.assigned / rider.delivered
        │
        ▼
Notification Service
        │
        ├── Classification Engine
        │   ├── Transactional? → FCM/HMS Push (high priority)
        │   ├── Engagement? → Braze Campaign
        │   └── Operational? → In-app notification
        │
        ├── Routing Engine
        │   ├── Device has Google Play? → FCM
        │   ├── Device has HMS only? → HMS Push Kit
        │   └── Both? → FCM (preferred)
        │
        ├── Delivery Engine
        │   ├── FCM: POST https://fcm.googleapis.com/fcm/send
        │   ├── HMS: POST https://push-api.cloud.huawei.com/v2/{appid}/messages:send
        │   └── Braze: via Braze SDK (client-side)
        │
        └── Tracking
            ├── Delivery receipts
            ├── Open rates
            └── Perseus analytics events
```

### 2.2 Braze Integration

Braze serves as the primary engagement and lifecycle management platform:

**Configuration (from decompiled TalabatApplication.java):**

| Parameter | Value |
|-----------|-------|
| Google Play API Key | `f880a0a8-df23-4a78-80ee-096cfd56ea67` |
| HMS API Key | `0d387798-0f0b-43b6-8610-fea0ce9fe7fc` |
| Custom Endpoint | `sdk.iad-01.braze.com` |
| Session Timeout | 60 seconds |
| Push Deep Links | Enabled, back stack → `FlutterLandingScreen` |
| Push HTML Rendering | Enabled |
| Auto Deep Link Handling | Enabled |
| Network Data Flush Interval | 10 seconds |

**Braze Features Used:**

| Feature | Implementation |
|---------|---------------|
| Push notifications | Via `BrazeFirebaseMessagingService` |
| In-app messages | Custom factory (`BrazeInAppMessageManager.setCustomInAppMessageViewFactory`) |
| In-app message listener | Custom listener (`setCustomInAppMessageManagerListener`) |
| Content cards | Likely used for promotional content on home screen |
| User attributes | First name, Pro status, country, wallet balance |
| Custom events | Order placed, cart abandoned, search performed |
| Deep linking | `setPushDeepLinkBackStackActivityEnabled(true)` |
| Image loading | Custom `BrazeImageLoader` via Flutter plugin |

---

## 3. Notification Channels

### 3.1 Android Notification Channels

The app creates 12 notification channels during Application.onCreate(), each with distinct importance and purpose:

| Channel ID | Name | Importance | Use Case |
|-----------|------|-----------|----------|
| `prospect` | Prospect | HIGH (224) | Prospective user acquisition and onboarding |
| `lifecycle_engagement` | Lifecycle Engagement | HIGH (224) | Re-engagement for dormant users |
| `churning_winback` | Churning Winback | HIGH (224) | Win-back campaigns for high-churn-risk users |
| `adhoc_restaurant_deals` | Restaurant Deals | HIGH (224) | Flash deals, restaurant-specific promotions |
| `adhoc_marketing_vouchers` | Marketing Vouchers | HIGH (224) | Voucher distribution and coupon campaigns |
| `abandonments` | Abandonments | HIGH (224) | Cart and checkout abandonment recovery |
| `app_feature_updates` | Feature Updates | HIGH (224) | New feature announcements and tips |
| `transactional` | Transactional | DEFAULT (128) | Order confirmations, payment receipts, delivery updates |
| `talabatgo` | talabat GO | HIGH (224) | TGO (Guaranteed Order) status updates |
| `brand_updates` | Brand Updates | HIGH (224) | Updates from followed/favorite brands |
| `general` | General | HIGH (224) | General announcements |
| `chat` | Chat | HIGH (224) | Customer-rider and customer-support messages |

**Channel Design Rationale:**

The **transactional** channel has lower importance (DEFAULT = 128) compared to marketing channels (HIGH = 224) because transactional notifications are expected and less urgent to interrupt the user. Marketing channels use HIGH importance to ensure visibility, but users can individually opt out of marketing channels while keeping transactional notifications enabled — a key compliance requirement.

### 3.2 iOS Notification Categories

While the analyzed APK is Android, the notification system is designed for cross-platform parity. iOS notification categories mirror Android channels with similar user control.

---

## 4. Notification Types

### 4.1 Transactional Notifications

These are critical, user-initiated notifications related to active orders and payments:

| Notification | Trigger | Channel | Deep Link |
|-------------|---------|---------|-----------|
| Order confirmed | Order placed successfully | `transactional` | `/orders/{id}/tracking` |
| Order preparing | Vendor accepts order | `transactional` | `/orders/{id}/tracking` |
| Order on the way | Rider picks up order | `transactional` | `/orders/{id}/tracking` |
| Order delivered | Delivery confirmed | `transactional` | `/orders/{id}` |
| Order cancelled | Order cancelled (any reason) | `transactional` | `/orders/{id}` |
| Payment confirmed | Payment captured | `transactional` | `/wallet` |
| Payment failed | Payment declined | `transactional` | `/checkout` |
| Refund processed | Refund completed | `transactional` | `/wallet` |
| BNPL payment due | 3 days before due date | `transactional` | `/bnpl` |
| BNPL payment overdue | Payment past due | `transactional` | `/bnpl` |
| Prescription approved | Pharmacy order approved | `transactional` | `/orders/{id}` |
| Item replacement needed | Out of stock during picking | `transactional` | `/orders/{id}` |
| Chat message | New rider/support message | `chat` | `/orders/{id}/chat` |

### 4.2 Engagement Notifications

These are marketing and lifecycle campaigns managed through Braze:

| Campaign Type | Targeting | Channel | Example |
|--------------|-----------|---------|---------|
| Prospecting | New app installs, no orders | `prospect` | "Your first order is waiting! Get 20% off" |
| Lifecycle | Users in specific lifecycle stages | `lifecycle_engagement` | "We miss you! Here's AED 15 off your next order" |
| Churn prevention | High churn risk (no order in 14+ days) | `churning_winback` | "Your favorite restaurant has a new deal!" |
| Flash deals | All users in delivery area | `adhoc_restaurant_deals` | "McDonald's: 30% off for the next 2 hours!" |
| Voucher campaigns | Segmented by spend behavior | `adhoc_marketing_vouchers` | "You've earned a AED 25 voucher!" |
| Abandonment recovery | Cart/checkout abandoners | `abandonments` | "Your cart is waiting! Complete your order now" |
| Feature updates | All users / new feature adopters | `app_feature_updates` | "Try our new AI chat to find your next meal!" |
| Brand updates | Users who favorited a brand | `brand_updates` | "Al Safadi just added new dishes!" |
| TGO promotions | Users with active/nearby orders | `talabatgo` | "Track your order with guaranteed delivery" |

### 4.3 In-App Notifications

In-app messages are delivered through Braze's in-app message framework with custom UI:

| Type | Trigger | Visual |
|------|---------|--------|
| Full-screen modal | Major announcements | Full screen with image and CTA |
| Slide-up | Minor promotions | Bottom slide-up banner |
| Modal | Targeted offers | Centered card with accept/dismiss |
| Custom HTML | Rich content | `braze-html-bridge.js` + `braze-html-in-app-message-bridge.js` for HTML5 in-app messages |

---

## 5. Push Notification Processing

### 5.1 TalabatFirebaseMessagingService

The `TalabatFirebaseMessagingService` extends `BrazeFirebaseMessagingService`, creating a dual-processing pipeline:

```kotlin
// Simplified from decompiled code
override fun onMessageReceived(remoteMessage: RemoteMessage) {
    // Step 1: Try Braze processing first
    if (BrazeFirebaseMessagingService.handleMessageReceived(remoteMessage)) {
        return // Braze handled this message
    }
    
    // Step 2: Custom Talabat processing
    val data = remoteMessage.data
    when (data["type"]) {
        "order_status_update" -> handleOrderNotification(data)
        "chat_message" -> handleChatNotification(data)
        "payment_update" -> handlePaymentNotification(data)
        "delivery_update" -> handleDeliveryNotification(data)
    }
    
    // Step 3: Update Live Activity if applicable
    LiveNotificationService.updateOrder(data)
}
```

### 5.2 Notification Display Logic

```
Push received
        │
        ├── App in foreground?
        │   ├── YES → Show in-app banner (heads-up) + sound
        │   └── NO → Show system notification + sound/vibration
        │
        ├── Notification channel enabled?
        │   ├── YES → Display notification
        │   └── NO → Suppress (respect user preference)
        │
        ├── Do Not Disturb mode?
        │   ├── Transactional → Still display (HIGH importance bypasses DND)
        │   └── Marketing → Suppress
        │
        └── Deep link present?
            ├── YES → Tap opens specific screen
            └── NO → Tap opens home screen
```

### 5.3 Notification Sound & Vibration

| Channel | Sound | Vibration | LED |
|---------|-------|-----------|-----|
| `transactional` | Default | Default | Orange |
| `chat` | Custom (soft chime) | Short | Orange |
| All marketing | Default | Default | None |
| `talabatgo` | Custom (distinct) | Pattern | Blue |

---

## 6. Live Activity / Foreground Tracking

### 6.1 Android Live Activity

The `LiveNotificationService` provides persistent, always-visible order tracking:

**Notification Layout:**
```
┌──────────────────────────────────────────┐
│  🏍️ Al Safadi · On the way              │
│  ETA: 12 minutes                         │
│  ┌──────────────────────────────────┐    │
│  │  [Track]    [Chat]    [Call]     │    │
│  └──────────────────────────────────┘    │
└──────────────────────────────────────────┘
```

**Lifecycle:**
1. Created when order enters "delivering" state
2. Updated with real-time ETA (via Firebase RTDB listener)
3. Shows rider actions (track, chat, call)
4. Auto-dismissed when order is delivered
5. Auto-expired after 2 hours if not delivered

### 6.2 iOS Live Activity

For iOS, the app likely uses Apple's Live Activity framework (ActivityKit) to show Dynamic Island and Lock Screen widgets with similar tracking information.

---

## 7. Notification Personalization

### 7.1 Braze User Attributes

The following user attributes are synced to Braze for personalization:

| Attribute | Type | Source | Usage |
|-----------|------|--------|-------|
| `first_name` | String | User profile | Personalization ("{customerName}, you're such a pro") |
| `country_code` | String | User profile | Country-specific campaigns |
| `pro_status` | String | Subscription service | Pro/non-Pro targeting |
| `last_order_date` | Date | Order service | Churn risk scoring |
| `total_orders` | Integer | Order service | Loyalty tier |
| `favorite_cuisines` | Array | Search/order history | Restaurant recommendations |
| `preferred_vertical` | String | Order history | Vertical-specific campaigns |
| `wallet_balance` | Float | Wallet service | Wallet top-up campaigns |
| `bnpl_available_limit` | Float | BNPL service | BNPL adoption campaigns |

### 7.2 Braze Custom Events

| Event | Properties | Trigger |
|-------|-----------|---------|
| `order_placed` | vendor_id, vertical, total | Order confirmed |
| `order_delivered` | vendor_id, delivery_time | Delivery completed |
| `cart_abandoned` | vendor_id, cart_value | Cart abandoned > 30 min |
| `search_performed` | query, vertical, results_count | Search executed |
| `voucher_applied` | voucher_code, discount | Voucher redeemed |
| `pro_subscribed` | plan_type | Pro subscription activated |
| `pro_cancelled` | reason | Pro subscription cancelled |
| `payment_failed` | method, error | Payment declined |
| `app_opened` | source, deep_link | App launched |

---

## 8. Notification Preferences

### 8.1 User Control

Users can manage notification preferences per channel:

| Channel | Default | User Can Disable |
|---------|---------|-----------------|
| Transactional | ON | No (regulatory requirement) |
| Chat | ON | Yes |
| Prospect | ON | Yes |
| Lifecycle Engagement | ON | Yes |
| Churning Winback | ON | Yes |
| Restaurant Deals | ON | Yes |
| Marketing Vouchers | ON | Yes |
| Abandonments | ON | Yes |
| Feature Updates | ON | Yes |
| TGO | ON | Yes |
| Brand Updates | ON | Yes |
| General | ON | Yes |

### 8.2 Smart Suppression Rules

| Rule | Condition | Action |
|------|-----------|--------|
| Daily cap | > 5 marketing notifications in 24h | Suppress remaining |
| Frequency cap | Same campaign > 1x in 24h | Suppress duplicate |
| Sleep hours | Between 11 PM - 8 AM (local) | Queue for morning delivery |
| Recent order | Order placed < 30 min ago | Suppress non-transactional |
| Uninstalled | Braze detects uninstall | Stop sending (bounce detection) |
| Inactive user | No app opens in 30 days | Reduce frequency |

---

## 9. Notification Analytics

### 9.1 Tracked Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| Delivery rate | Notifications delivered / sent | > 95% |
| Open rate | Notifications opened / delivered | > 15% (marketing), > 80% (transactional) |
| Click-through rate | CTA clicked / opened | > 30% (marketing) |
| Conversion rate | Target action completed / clicked | > 5% (marketing) |
| Opt-out rate | Users disabling channel / total users | < 2% |
| Bounce rate | Failed deliveries / sent | < 3% |

### 9.2 Perseus Event Tracking

| Event | Properties |
|-------|-----------|
| `notification_received` | type, channel, campaign_id |
| `notification_opened` | type, channel, campaign_id, deep_link |
| `notification_dismissed` | type, channel, campaign_id |
| `notification_action_taken` | type, channel, action, campaign_id |
| `in_app_message_shown` | message_id, campaign_id |
| `in_app_message_clicked` | message_id, campaign_id, button_id |

---

## 10. Compliance & Regulations

### 10.1 Opt-In Requirements

| Region | Requirement | Implementation |
|--------|-------------|----------------|
| All markets | Explicit consent for marketing | Opt-in checkbox during registration (`authentication.sign_up.offers.check.box.title`) |
| UAE | TRA regulations | Unsubscribe option in every marketing notification |
| Saudi Arabia | CITC regulations | Arabic-language opt-out instructions |
| Egypt | NTRA regulations | Local data processing requirements |
| GDPR-equivalent | Right to withdraw consent | Per-channel notification settings |

### 10.2 Data Processing

- **Braze data residency**: Braze processes data in US (`sdk.iad-01.braze.com`)
- **AI chat disclaimer**: Explicit disclosure of OpenAI data transfer to US (`ai_chat_disclaimer`)
- **Incognia**: Location data collected only with consent, installed apps collection disabled
- **Data retention**: Notification logs retained for 90 days for analytics
