# 11 — Real-Time Architecture

## 1. Overview

Talabat's real-time architecture enables live, bidirectional communication between the mobile client, backend services, and operational systems (riders, vendors). The system powers critical features including live order tracking with rider GPS position updates, real-time order status notifications, in-app chat between customers and riders, and dynamic ETA recalculation. The architecture must deliver sub-second latency for tracking updates while gracefully handling network transitions (WiFi to cellular, dead zones) and device sleep states.

The real-time layer is built on three primary technologies: **Firebase Realtime Database** (for order tracking synchronization), **Firebase Cloud Messaging / HMS Push** (for notification delivery), and **OkHttp WebSocket** (available for future bidirectional communication). The system is designed with an **optimistic update** approach where the client assumes success and reconciles with server state on reconnection, providing a smooth user experience even under poor network conditions.

---

## 2. Real-Time Communication Channels

### 2.1 Channel Overview

| Channel | Technology | Direction | Latency | Use Case |
|---------|-----------|-----------|---------|----------|
| Order Tracking | Firebase RTDB | Server → Client | < 1s | Rider location, status updates |
| Push Notifications | FCM / HMS Push | Server → Client | < 3s | Order status, marketing, engagement |
| In-App Chat | Firebase RTDB | Bidirectional | < 2s | Customer ↔ Rider messaging |
| Live Activity | Android Foreground Service | Local | Immediate | Persistent tracking notification |
| SSE (Server-Sent Events) | HTTP/2 SSE | Server → Client | < 2s | Menu availability, search results |
| Analytics Events | HTTP POST (batched) | Client → Server | ~30s batch | Perseus event tracking |

### 2.2 Technology Selection Rationale

| Technology | Why Chosen | Limitations |
|-----------|-----------|-------------|
| Firebase RTDB | Real-time sync with offline support; native Flutter plugin; automatic reconnection | Cost at scale; limited query capabilities; vendor lock-in |
| FCM/HMS Push | Industry standard; reliable delivery; dual-platform support (Google + Huawei) | Not guaranteed delivery; no backpressure control |
| OkHttp WebSocket | Available in codebase; low latency; bidirectional | Not currently used for primary flows |
| SSE | Simple HTTP-based; works with CDN; HTTP/2 multiplexing | Unidirectional only; limited browser support (not relevant for mobile) |

---

## 3. Firebase Realtime Database Architecture

### 3.1 Data Structure

```
talabat-tracking/
├── order_tracking/
│   └── {order_id}/
│       ├── status/
│       │   ├── current: "delivering"          # Current order status
│       │   ├── updated_at: 1700000000000      # Timestamp (ms)
│       │   └── estimated_arrival: 1700001200  # Unix timestamp
│       ├── rider/
│       │   ├── id: "rider_uuid"
│       │   ├── name: "Muhammad"
│       │   ├── phone: "+971509876543"
│       │   ├── vehicle_type: "motorcycle"
│       │   ├── rating: 4.8
│       │   └── photo_url: "https://..."
│       ├── location/
│       │   ├── latitude: 25.0785              # Updated every 5-10s
│       │   ├── longitude: 55.1385
│       │   ├── heading: 180.5                 # Degrees (0-360)
│       │   ├── speed: 35.2                    # km/h
│       │   ├── accuracy: 10.0                 # Meters
│       │   └── updated_at: 1700000000000
│       └── chat/
│           └── {message_id}/
│               ├── sender_type: "customer"    # customer | rider | support
│               ├── sender_name: "Ahmed"
│               ├── message_text: "I'm at the gate"
│               ├── image_url: null
│               ├── is_read: false
│               └── sent_at: 1700000000000
```

### 3.2 Firebase RTDB Connection Management

```
Flutter App (firebase_database plugin)
        │
        ├── FirebaseDatabasePlugin
        │   ├── Maintains persistent WebSocket to Firebase
        │   ├── Automatic reconnection on network change
        │   ├── Offline disk persistence enabled
        │   └── Connection state listener
        │
        ├── Authentication
        │   ├── Firebase Auth with custom token (from Talabat backend)
        │   ├── Token refresh handled automatically
        │   └── Security rules enforce user-specific access
        │
        └── Listeners
            ├── ValueEventListener on order_tracking/{order_id}/status
            ├── ValueEventListener on order_tracking/{order_id}/location
            └── ChildEventListener on order_tracking/{order_id}/chat
```

### 3.3 Firebase Security Rules

```json
{
  "rules": {
    "order_tracking": {
      "$order_id": {
        ".read": "auth != null && ($order_id in root.child('user_orders').child(auth.uid).val().split(','))",
        ".write": false,
        "location": {
          ".read": "auth != null && isCustomerOfOrder($order_id)",
          ".write": "auth != null && isAssignedRider($order_id)"
        },
        "chat": {
          ".read": "auth != null && (isCustomerOfOrder($order_id) || isAssignedRider($order_id))",
          ".write": "auth != null && (isCustomerOfOrder($order_id) || isAssignedRider($order_id))"
        }
      }
    }
  }
}
```

### 3.4 Offline Behavior

Firebase RTDB provides built-in offline support:

| Scenario | Behavior |
|----------|----------|
| Network lost during tracking | Client continues showing last known rider position |
| Network restored | Firebase automatically syncs to latest state |
| App backgrounded | Firebase connection maintained via Android Service |
| App killed | Reconnection on next app launch; missed updates reconciled |
| Disk persistence | Last 24 hours of tracking data cached on device |

---

## 4. Push Notification Architecture

### 4.1 FCM / HMS Push Integration

The app implements **dual push notification support** to cover both Google Play Services and Huawei Mobile Services devices:

```
Backend Notification Service
        │
        ├── Check device type
        │   ├── Google Play Services → FCM
        │   │   POST https://fcm.googleapis.com/fcm/send
        │   │   Authorization: key=...
        │   │
        │   └── HMS Push → Huawei Push Kit
        │       POST https://push-api.cloud.huawei.com/v2/{appid}/messages:send
        │       Authorization: Bearer ...
        │
        └── Braze (marketing/engagement)
            Braze API → handles segmentation and delivery
```

### 4.2 Notification Data Structure

**FCM Message Format:**
```json
{
  "to": "device_token",
  "notification": {
    "title": "Your order is on the way!",
    "body": "Muhammad is delivering your order from Al Safadi",
    "sound": "default",
    "click_action": "ORDER_TRACKING"
  },
  "data": {
    "type": "order_status_update",
    "order_id": "uuid",
    "status": "delivering",
    "deep_link": "talabat://orders/uuid/tracking",
    "channel_id": "transactional"
  },
  "android": {
    "notification": {
      "channel_id": "transactional",
      "icon": "ic_notification",
      "color": "#FF5A00"
    },
    "priority": "high"
  }
}
```

### 4.3 TalabatFirebaseMessagingService

The custom `TalabatFirebaseMessagingService` extends `BrazeFirebaseMessagingService`, creating a layered notification processing pipeline:

```
FCM Message Received
        │
        ▼
BrazeFirebaseMessagingService.onMessageReceived()
        │
        ├── Is Braze push? → Process via Braze SDK
        │   ├── In-app message → Show via BrazeInAppMessageManager
        │   ├── Push campaign → Show via Braze notification builder
        │   └── Deep link → Route via Braze deep link handler
        │
        └── Not Braze push → Talabat custom processing
            ├── Order notification → Update Live Activity + Show notification
            ├── Chat message → Show chat notification + Update RTDB listener
            ├── Marketing → Show in appropriate channel
            └── System → Show transactional notification
```

### 4.4 Notification Channels

| Channel ID | Name | Importance | Use Case |
|-----------|------|-----------|----------|
| `prospect` | Prospect | HIGH (224) | Prospective user engagement |
| `lifecycle_engagement` | Lifecycle Engagement | HIGH (224) | Re-engagement campaigns |
| `churning_winback` | Churning Winback | HIGH (224) | Win-back campaigns for churned users |
| `adhoc_restaurant_deals` | Restaurant Deals | HIGH (224) | Flash deals and restaurant promotions |
| `adhoc_marketing_vouchers` | Marketing Vouchers | HIGH (224) | Voucher and coupon campaigns |
| `abandonments` | Abandonments | HIGH (224) | Cart/order abandonment reminders |
| `app_feature_updates` | Feature Updates | HIGH (224) | New feature announcements |
| `transactional` | Transactional | DEFAULT (128) | Order status, payment confirmations |
| `talabatgo` | talabat GO | HIGH (224) | TGO (Guaranteed Order) updates |
| `brand_updates` | Brand Updates | HIGH (224) | Updates from followed brands |
| `general` | General | HIGH (224) | General notifications |
| `chat` | Chat | HIGH (224) | Rider/customer chat messages |

---

## 5. Live Activity (Android)

### 5.1 LiveNotificationService

The `LiveNotificationService` (`com.talabat.live_activity_plugin`) provides persistent order tracking on Android as a foreground service:

```
LiveNotificationService (Foreground Service)
        │
        ├── onCreate()
        │   ├── Create foreground notification with order status
        │   ├── Load active_orders from SharedPreferences
        │   ├── Listen to Firebase RTDB for each active order
        │   └── Start location updates (if needed)
        │
        ├── onOrderStatusUpdate(order_id, status)
        │   ├── Update foreground notification content
        │   ├── Update notification with: vendor name, status, ETA
        │   ├── If delivered: Remove order from active_orders
        │   └── If no active orders: Stop foreground service
        │
        ├── onRiderLocationUpdate(order_id, lat, lng)
        │   ├── Update notification with distance/direction
        │   └── Recalculate ETA
        │
        ├── Auto-Expiry
        │   ├── Orders auto-expire after 2 hours (7,200,000 ms)
        │   └── Service restarts restore state from SharedPreferences
        │
        └── onDestroy()
            ├── Save active_orders to SharedPreferences
            └── Clean up Firebase RTDB listeners
```

### 5.2 Android 14+ Foreground Service Types

The service declares appropriate foreground service types for Android 14+:

- `location` — For rider GPS tracking
- `dataSync` — For Firebase RTDB synchronization
- `specialUse` — For order tracking functionality

### 5.3 Service Lifecycle

```
Order Placed → Service Started
        │
        ├── Active Orders > 0: Service Running
        │   ├── Foreground notification visible
        │   ├── Firebase RTDB listeners active
        │   └── ETA updates every 30s
        │
        ├── All Orders Delivered/Expired: Service Stopped
        │   └── Foreground notification dismissed
        │
        ├── App Killed: Service Persists (foreground service)
        │   └── Restores from SharedPreferences on next launch
        │
        └── Device Restart: Service Not Auto-Restarted
            └── Re-activated when app is opened and active orders exist
```

---

## 6. Real-Time Order Status Flow

### 6.1 Status Update Pipeline

```
Vendor accepts order
        │
        ▼
Order Service → Kafka: order.status_changed (ordered → preparing)
        │
        ├── Notification Service → FCM/HMS: Push to customer
        ├── Dispatch Service → Firebase RTDB: Update status
        │       └── Customer app listener fires → UI update
        ├── Live Activity Service → Update foreground notification
        └── Perseus Analytics → Track event
```

### 6.2 Rider Location Update Pipeline

```
Rider App GPS (every 5-10 seconds)
        │
        ▼
Rider Location Service
        │
        ├── Redis: SET rider:{id}:location {lat, lng, heading, speed} EX 10
        │
        ├── Firebase RTDB: SET order_tracking/{order_id}/location
        │       └── Customer app listener fires → Map marker update
        │
        ├── Dispatch Service
        │   ├── Recalculate ETA
        │   ├── Update order_eta in Redis
        │   └── If ETA change > 5 min → Push notification
        │
        └── MongoDB: INSERT rider_locations (analytics)
```

### 6.3 Chat Message Flow

```
Customer types message
        │
        ▼
Firebase RTDB: PUSH order_tracking/{order_id}/chat/{new_id}
        │
        ├── Rider app listener fires → Display message
        ├── Push notification to rider (if app backgrounded)
        └── Perseus: Track chat_sent event

Rider types response
        │
        ▼
Firebase RTDB: PUSH order_tracking/{order_id}/chat/{new_id}
        │
        ├── Customer app listener fires → Display message
        ├── Push notification via chat channel
        └── Update unread count badge
```

---

## 7. Network Resilience

### 7.1 Offline Handling Strategy

| Feature | Offline Behavior | Reconnection Behavior |
|---------|-----------------|----------------------|
| Order tracking | Show last known rider position | Auto-sync to latest position |
| Order status | Show last known status | Update to current status |
| Chat | Queue message locally, show as "sending" | Send queued messages, mark as sent |
| Menu browsing | Serve from local cache (BAE) | Refresh cache from API |
| Cart operations | Queue operations locally | Replay operations on reconnection |
| Search | Show cached results | Execute search against API |

### 7.2 Network Quality Adaptation

| Condition | Adaptation |
|-----------|------------|
| No network | Full offline mode with cached data |
| Poor network (2G/slow) | Reduce image quality, disable video, batch API calls |
| Good network (4G/WiFi) | Full feature set, HD images, video stories |
| Network transition | Automatic reconnection with exponential backoff |

### 7.3 Connection State Monitoring

The `connectivity_plus` plugin (`dev.fluttercommunity.plus.connectivity`) monitors network state changes:

- WiFi → Cellular: Seamless transition, Firebase reconnects
- Online → Offline: Show offline banner, switch to cached data
- Offline → Online: Auto-sync, refresh stale data

---

## 8. Real-Time Performance

### 8.1 Latency Targets

| Operation | Target Latency | Measurement Point |
|-----------|---------------|-------------------|
| Rider location update (Rider → Customer) | < 2s | GPS → Firebase RTDB → Client listener |
| Order status change (Server → Customer) | < 3s | Kafka → Firebase RTDB → Client listener |
| Chat message delivery | < 2s | Sender → Firebase RTDB → Receiver listener |
| Push notification delivery | < 5s | Backend → FCM → Device display |
| ETA recalculation | < 5s | Location update → ETA engine → Client update |

### 8.2 Throughput Requirements

| Metric | Peak (Ramadan iftar) | Normal |
|--------|---------------------|--------|
| Active tracked orders | 100,000+ | 30,000 |
| Rider location updates/second | 10,000+ | 3,000 |
| Chat messages/minute | 5,000+ | 1,500 |
| Push notifications/minute | 50,000+ | 15,000 |

### 8.3 Firebase RTDB Optimization

- **Shallow reads**: Only listen to specific paths, not entire order tree
- **Listener management**: Detach listeners when user leaves tracking screen
- **Batch writes**: Combine multiple updates into a single `updateChildren()` call
- **Connection pooling**: Single Firebase connection shared across all listeners
- **Data denormalization**: Rider location stored separately from order metadata to minimize sync payload

---

## 9. Future Architecture Considerations

### 9.1 WebSocket Migration Path

The OkHttp WebSocket library (`okhttp3.WebSocket`) is bundled in the app but not currently used for primary real-time communication. A potential migration path could replace Firebase RTDB with a custom WebSocket server for:

- Reduced latency (direct server → client, no Firebase intermediary)
- Cost reduction at scale (Firebase RTDB pricing based on connections and data downloaded)
- Greater control over connection management and backpressure
- Protocol flexibility (binary protocols, compression)

### 9.2 HTTP/2 Server Push

The feature flag `exp_platform_http2` indicates experimentation with HTTP/2, which could enable server push for real-time updates without WebSocket infrastructure, particularly for one-way data flows like menu availability changes.

### 9.3 GraphQL Subscriptions

For future real-time features, GraphQL subscriptions over WebSocket could provide a typed, flexible real-time API that reduces the need for multiple Firebase RTDB listeners.
