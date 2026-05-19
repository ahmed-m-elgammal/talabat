# 05 — Real-Time Architecture Considerations

## 1. Overview

This plan addresses the real-time architecture for the Talabat-like MVP, based on the production real-time specifications in `docs/11_realtime_architecture.md`. Real-time capabilities are a core differentiator for any delivery marketplace — customers expect live order tracking, instant status updates, and responsive communication with riders. The MVP must deliver these experiences with sub-second latency for tracking and sub-5-second latency for status notifications, even under poor network conditions.

The MVP real-time architecture uses three primary channels, matching the production system's approach but simplified for single-country deployment:

1. **Firebase Realtime Database** — Order tracking (rider location, status updates)
2. **Firebase Cloud Messaging (FCM)** — Push notifications (order status, chat)
3. **HTTP Polling (fallback)** — When push/unreal-time channels are unavailable

---

## 2. Real-Time Communication Channels (MVP)

| Channel | Technology | Direction | Latency Target | Use Case |
|---------|-----------|-----------|---------------|----------|
| Order Tracking | Firebase RTDB | Server → Client | < 2s | Rider location, order status |
| Push Notifications | FCM | Server → Client | < 5s | Order status, payment, chat |
| In-App Chat | Firebase RTDB | Bidirectional | < 3s | Customer ↔ Rider messaging |
| Location Updates | HTTP POST + Firebase RTDB | Client → Server → Client | < 3s | Rider GPS position |
| Menu Availability | HTTP Polling (30-60s) | Client → Server | 30-60s | Stock status during browsing |

### 2.1 What's Excluded from MVP

| Channel | Production Technology | Why Excluded |
|---------|----------------------|-------------|
| HMS Push | Huawei Push Kit | Single-platform (Google Play) for MVP |
| SSE (Server-Sent Events) | HTTP/2 SSE | Polling is sufficient for menu availability at MVP scale |
| WebSocket | OkHttp WebSocket | Firebase RTDB provides equivalent functionality |
| Live Activity (iOS) | ActivityKit | Android-only for MVP; add iOS in Phase 2 |
| Braze In-App Messages | Braze SDK | No marketing automation platform for MVP |

---

## 3. Firebase RTDB Architecture (MVP)

### 3.1 Data Structure

```
talabat-tracking/
├── order_tracking/
│   └── {order_id}/
│       ├── status/
│       │   ├── current: "delivering"
│       │   ├── updated_at: 1700000000000
│       │   └── estimated_arrival: 1700001200
│       ├── rider/
│       │   ├── id: "rider_uuid"
│       │   ├── name: "Muhammad"
│       │   ├── phone: "+971509876543"
│       │   ├── vehicle_type: "motorcycle"
│       │   └── rating: 4.8
│       ├── location/
│       │   ├── latitude: 25.0785
│       │   ├── longitude: 55.1385
│       │   ├── heading: 180.5
│       │   ├── speed: 35.2
│       │   └── updated_at: 1700000000000
│       └── chat/
│           └── {message_id}/
│               ├── sender_type: "customer"
│               ├── message_text: "I'm at the gate"
│               ├── is_read: false
│               └── sent_at: 1700000000000
```

### 3.2 Security Rules

```json
{
  "rules": {
    "order_tracking": {
      "$order_id": {
        ".read": "auth != null",
        "status": {
          ".read": "auth != null",
          ".write": "auth != null && (auth.token.role === 'system' || auth.token.role === 'vendor')"
        },
        "rider": {
          ".read": "auth != null",
          ".write": "auth != null && auth.token.role === 'system'"
        },
        "location": {
          ".read": "auth != null",
          ".write": "auth != null && auth.token.role === 'rider'"
        },
        "chat": {
          ".read": "auth != null",
          ".write": "auth != null"
        }
      }
    }
  }
}
```

### 3.3 Firebase Auth (Custom Tokens)

The backend generates Firebase custom tokens for authenticated users:

```
Customer login → Backend generates Firebase custom token with:
  - uid: user_id
  - claims: { role: "customer", order_ids: ["uuid1", "uuid2"] }

Rider login → Backend generates Firebase custom token with:
  - uid: rider_id
  - claims: { role: "rider", assigned_order_ids: ["uuid3"] }
```

---

## 4. Tasks

### 4.1 Tracking Tasks

---

#### Task RT-TRACK-01: Firebase RTDB Setup & Security Rules
**Description**: Set up Firebase project with RTDB, configure security rules per the data structure above, implement custom token generation on the backend, and test rule enforcement.
**Dependencies**: B-INFRA-02
**Acceptance Criteria**:
- Firebase project created with RTDB in target region
- Security rules deployed and tested: customer reads own orders, rider writes location, system writes status
- Custom token generation endpoint on backend
- Token includes: uid, role, and relevant order assignments
- Token refresh handled (custom tokens expire in 1 hour)
- RTDB persistence enabled for offline support on client

---

#### Task RT-TRACK-02: Rider Location Update Pipeline
**Description**: Implement the rider location update flow. Rider app sends GPS coordinates every 5-10 seconds → backend writes to Firebase RTDB → customer app listens for updates → map marker moves in real-time.
**Dependencies**: RT-TRACK-01, B-DISP-01
**Acceptance Criteria**:
- Rider app sends location updates via `POST /rider/location` or directly to Firebase RTDB
- Updates written to `order_tracking/{order_id}/location` in RTDB
- Customer app `ValueEventListener` fires on location change
- Map marker position updated on customer tracking screen
- Location update latency: < 2s from GPS read to map update
- Location updates stop when rider goes offline or order delivered
- Last known position shown when updates stop (no disappearing marker)

---

#### Task RT-TRACK-03: Order Status Real-Time Updates
**Description**: Implement real-time order status updates. When order status changes on backend (vendor accepts, rider picks up, etc.), update Firebase RTDB and send FCM push notification. Customer app listens to RTDB for instant UI update.
**Dependencies**: RT-TRACK-01, B-ORDER-02
**Acceptance Criteria**:
- Status change on backend → update `order_tracking/{order_id}/status/current` in RTDB
- Customer app listener fires → UI updates instantly (no polling)
- FCM push notification sent simultaneously as fallback
- Status change events: ordered, preparing, delivering, delivered, cancelled
- Status timeline visible on order tracking screen
- Home screen active order card updates in real-time

---

#### Task RT-TRACK-04: ETA Recalculation & Display
**Description**: Implement dynamic ETA display on tracking screen. ETA recalculated every 30 seconds based on rider location, remaining distance, and average speed. Updated ETA written to Firebase RTDB.
**Dependencies**: RT-TRACK-02
**Acceptance Criteria**:
- ETA engine runs on backend, recalculates every 30 seconds for active orders
- ETA based on: remaining distance / average rider speed
- For MVP: simple distance/speed calculation (no traffic API integration)
- Updated ETA written to `order_tracking/{order_id}/status/estimated_arrival`
- Customer app displays ETA range: "25-35 min" during preparation, narrows near delivery
- ETA change > 5 min triggers additional push notification

---

### 4.2 Push Notification Tasks

---

#### Task RT-PUSH-01: FCM Integration & Device Token Management
**Description**: Integrate Firebase Cloud Messaging for push notifications. Register device tokens on login, unregister on logout. Handle foreground and background message reception.
**Dependencies**: RT-TRACK-01
**Acceptance Criteria**:
- FCM token obtained on app launch and sent to backend
- Token refreshed on FCM's `onTokenRefresh` callback
- Tokens stored in database: `device_tokens` table with user_id, token, platform
- Duplicate tokens handled (same device, same token)
- Tokens cleaned up on user logout
- Push received in foreground: show in-app banner
- Push received in background: show system notification
- Tap on notification: deep link to relevant screen (order tracking, etc.)

---

#### Task RT-PUSH-02: Transactional Push Notifications
**Description**: Implement transactional push notifications for all order lifecycle events. Each notification includes a deep link to the relevant screen.
**Dependencies**: RT-PUSH-01, B-NOTIF-01
**Acceptance Criteria**:
- Notifications sent for: order confirmed, preparing, on the way, delivered, cancelled
- Each notification includes: title, body, deep link, order_id
- Notification channel: "transactional" (DEFAULT importance)
- Deep link opens order tracking screen for order-related notifications
- No more than 1 notification per status change (no duplicates)
- Failed deliveries retried once after 5 minutes
- Notifications logged for analytics

---

#### Task RT-PUSH-03: Chat Push Notifications
**Description**: Implement push notifications for rider-customer chat messages. When a chat message is sent, push notification delivered to the other party if their chat screen is not active.
**Dependencies**: RT-PUSH-01, RT-CHAT-01
**Acceptance Criteria**:
- New chat message → push notification to recipient
- If recipient has chat screen open: no push (handled by RTDB listener)
- Notification: "{Sender}: {message_text}" with deep link to chat
- Notification channel: "chat" (HIGH importance)
- Badge count updated on app icon
- Tap opens chat screen for the specific order

---

### 4.3 Chat Tasks

---

#### Task RT-CHAT-01: Real-Time Chat via Firebase RTDB
**Description**: Implement in-app chat between customer and rider using Firebase RTDB. Messages stored in `order_tracking/{order_id}/chat/`. Both parties listen for new messages via `ChildEventListener`.
**Dependencies**: RT-TRACK-01
**Acceptance Criteria**:
- Chat screen shows message history for the order
- New messages appear in real-time (< 3s latency)
- Predefined quick messages: "I'm at the gate", "Come to lobby", "Leave at door"
- Custom text message input
- Message status: sent (single check), delivered (double check)
- Timestamps on each message
- Chat disabled after order delivered (read-only history)
- Offline: messages queued locally and sent on reconnection
- Firebase RTDB offline persistence handles message buffering

---

### 4.4 Offline & Resilience Tasks

---

#### Task RT-OFF-01: Offline Handling Strategy
**Description**: Implement comprehensive offline handling following the production patterns from `docs/11_realtime_architecture.md` Section 7. The app must gracefully handle network transitions, provide visual feedback, and resume real-time updates on reconnection.
**Dependencies**: RT-TRACK-01, RT-PUSH-01
**Acceptance Criteria**:
- Network state monitored via `connectivity_plus` plugin
- Offline banner shown at top of screen when disconnected
- Order tracking: last known rider position shown, "Reconnecting..." indicator
- Chat: messages queued locally, marked "sending", delivered on reconnect
- Cart: operations queued locally, replayed on reconnect
- Menu browsing: cached data served with "Showing cached data" toast
- Firebase RTDB: automatic reconnection on network restore
- All pending operations replayed in order on reconnection
- No data loss during network transitions (WiFi → cellular, dead zones)

---

#### Task RT-OFF-02: Firebase RTDB Offline Persistence
**Description**: Enable Firebase RTDB disk persistence on the client. This allows the app to serve the last known tracking state even when launched offline.
**Dependencies**: RT-TRACK-01
**Acceptance Criteria**:
- `FirebaseDatabase.instance.setPersistenceEnabled(true)` called on app init
- Last 24 hours of tracking data cached on device
- App launched offline → last known order status and rider position shown
- "Last updated {time}" indicator on offline tracking data
- Automatic sync to latest state when connection restored
- Cache size bounded (prevent excessive disk usage)

---

### 4.5 Performance Tasks

---

#### Task RT-PERF-01: Real-Time Performance Monitoring
**Description**: Implement performance tracking for real-time operations. Measure latency from event source to UI update. Track Firebase RTDB connection state and reconnection times.
**Dependencies**: RT-TRACK-01, RT-PUSH-01
**Acceptance Criteria**:
- Rider location update latency tracked: GPS → RTDB write → Client listener → Map update
- Order status update latency tracked: Status change → RTDB write → Client listener → UI update
- Push notification delivery latency tracked: Backend → FCM → Device display
- Firebase RTDB connection state monitored: connected, disconnected, reconnecting
- Performance metrics reported to analytics (simplified Perseus equivalent)
- Alert on p99 latency exceeding targets: tracking > 3s, push > 10s

---

#### Task RT-PERF-02: Firebase RTDB Optimization
**Description**: Optimize Firebase RTDB usage to minimize costs and maximize performance. Follow the production optimization patterns from `docs/11_realtime_architecture.md` Section 8.3.
**Dependencies**: RT-TRACK-01
**Acceptance Criteria**:
- Shallow reads: listeners on specific paths only (`/status`, `/location`, `/chat`), not entire order tree
- Listener management: detach listeners when leaving tracking screen
- Batch writes: combine multiple updates into single `updateChildren()` call
- Data denormalization: rider location stored separately from order metadata
- Connection sharing: single Firebase connection across all listeners
- Bandwidth monitoring: track data downloaded per session

---

## 5. Real-Time Data Flow Diagrams

### 5.1 Order Status Update Flow

```
Vendor accepts order
        │
        ▼
Backend Order Service
        ├── PostgreSQL: UPDATE orders SET status='preparing'
        ├── Internal Event Bus: PUBLISH order.status_changed
        │       │
        │       ▼
        │   Notification Module
        │       ├── Firebase RTDB: SET order_tracking/{id}/status/current = "preparing"
        │       └── FCM: SEND push notification to customer
        │
        └── Analytics: TRACK event
                │
                ▼
Customer App
        ├── RTDB listener fires → UI updates (tracking screen, home card)
        └── Push notification received (if app backgrounded)
```

### 5.2 Rider Location Update Flow

```
Rider App GPS (every 5-10 seconds)
        │
        ├── HTTP POST /rider/location {lat, lng, heading, speed}
        │       │
        │       ▼
        │   Backend Dispatch Module
        │       ├── Redis: SET rider:{id}:location {lat, lng} EX 10
        │       ├── Firebase RTDB: SET order_tracking/{id}/location
        │       └── ETA Engine: recalculate if needed
        │
        └── (Alternative) Direct Firebase RTDB write from rider app
                │
                ▼
Customer App
        ├── RTDB listener fires → Map marker moves
        ├── ETA display updates
        └── Home card ETA updates
```

### 5.3 Chat Message Flow

```
Customer types message
        │
        ▼
Firebase RTDB: PUSH order_tracking/{id}/chat/{new_id}
        │
        ├── Rider app listener fires → Display message
        └── Push notification to rider (if app backgrounded)

Rider types response
        │
        ▼
Firebase RTDB: PUSH order_tracking/{id}/chat/{new_id}
        │
        ├── Customer app listener fires → Display message
        └── Push notification to customer (if chat not active)
```

---

## 6. Real-Time Performance Targets

| Operation | Target Latency | Measurement |
|-----------|---------------|-------------|
| Rider location update (Rider → Customer) | < 2s | GPS → RTDB → Client listener → Map |
| Order status change (Server → Customer) | < 3s | Kafka → RTDB → Client listener → UI |
| Chat message delivery | < 3s | Sender → RTDB → Receiver listener → Display |
| Push notification delivery | < 5s | Backend → FCM → Device display |
| ETA recalculation | < 5s | Location update → ETA engine → Client |

---

## 7. Cost Projections for Real-Time Infrastructure

| Component | Monthly Cost (100-300 orders/day) | Monthly Cost (5,000+ orders/day) |
|-----------|----------------------------------|----------------------------------|
| Firebase RTDB (Blaze plan) | $50-100 | $500-2,000 |
| FCM | Free | Free |
| Redis (location cache) | Included in Redis instance | $30-60 |
| SMS for OTP | $50-200 | $500-2,000 |
| **Total Real-Time Cost** | **$100-300** | **$1,030-4,060** |

### 7.1 Firebase RTDB Cost Management

Firebase RTDB charges by data downloaded and stored. Key cost control strategies:

1. **Listener granularity**: Only listen to specific paths, not entire order trees
2. **Location update throttling**: Rider app sends updates every 5-10 seconds (not every second)
3. **Listener lifecycle**: Detach listeners when leaving tracking screen
4. **Data pruning**: Delete completed order tracking data after 24 hours
5. **Batch updates**: Combine multiple writes into single `updateChildren()` call

### 7.2 Migration Path (Phase 2)

When Firebase RTDB costs become significant (> $500/month), migrate to a custom WebSocket server:

```
Phase 1 (MVP): Firebase RTDB
        │
        ▼
Phase 2 (Growth): Custom WebSocket server (Socket.io / ws)
        │   - Reduced latency (no Firebase intermediary)
        │   - Cost control (no per-GB pricing)
        │   - Greater flexibility
        │
        ▼
Phase 3 (Scale): WebSocket cluster + Redis pub/sub
            - Horizontal scaling
            - Message persistence
            - Protocol flexibility
```

---

## 8. Testing Strategy for Real-Time Features

### 8.1 Real-Time Test Scenarios

| Scenario | Test Method | Expected Behavior |
|----------|-----------|-------------------|
| Normal tracking flow | Integration test: place order → track → deliver | Rider moves on map, status updates, ETA changes |
| Network loss during tracking | Disable network on client | Last known position shown, "Reconnecting..." indicator |
| Network restore | Re-enable network | Auto-sync to latest state within 5 seconds |
| App backgrounded during tracking | Send app to background | Push notifications received for status changes |
| App killed during tracking | Force kill app | Re-open shows current tracking state from RTDB |
| Chat while offline | Send chat while disconnected | Message queued locally, sent on reconnect |
| Multiple active orders | Place 2 orders simultaneously | Both tracked independently, separate map markers |
| Rider goes offline | Rider app loses connection | Last known position frozen, "Rider temporarily offline" message |

### 8.2 Load Testing

| Test | Target | Tool |
|------|--------|------|
| Concurrent Firebase connections | 200 simultaneous listeners | Firebase emulator + custom script |
| Location update throughput | 50 updates/second | Simulated rider GPS stream |
| Push notification delivery | 100 notifications in 10 seconds | Backend trigger + FCM |
| Chat message throughput | 20 messages/second | Simulated chat clients |
