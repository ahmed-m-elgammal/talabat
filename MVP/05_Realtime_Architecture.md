# 05 — Real-Time Architecture

## Overview

The MVP real-time architecture powers two critical features: **live order tracking** (rider GPS position on a map) and **order status updates** (preparing, on the way, delivered). The full Talabat architecture (see `docs/11_realtime_architecture.md`) uses Firebase RTDB for tracking, FCM/HMS dual push, OkHttp WebSocket, SSE, and a sophisticated live activity foreground service. The MVP simplifies this to Firebase RTDB for tracking and FCM for push notifications, which is sufficient for 50–100 concurrent tracked orders.

---

## MVP Real-Time Channels

### T5.1 — Firebase RTDB Integration

**Description:** Set up Firebase Realtime Database for live order tracking. The rider app writes GPS coordinates to RTDB; the customer app listens for changes and updates the map marker in real-time. Also used for order status updates as an alternative to polling.

**Dependencies:** T1.5 (dispatch service), T2.6 (order tracking screen).

**Acceptance Criteria:**
- [ ] Firebase project created with RTDB instance in the target region (e.g., me-central1 for UAE)
- [ ] Data structure: `order_tracking/{order_id}/status/` and `order_tracking/{order_id}/location/`
- [ ] Rider app writes location every 5–10 seconds: `{latitude, longitude, heading, speed, updated_at}`
- [ ] Backend writes status changes: `{current, estimated_arrival, updated_at}`
- [ ] Customer app attaches `ValueEventListener` on order_tracking/{order_id} when tracking screen is opened
- [ ] Listener detached when user leaves tracking screen (prevents unnecessary connections)
- [ ] Firebase Auth with custom token from Talabat backend (issued after order creation)
- [ ] Security rules: customers can only read their own order tracking data; riders can write location for assigned orders
- [ ] Offline disk persistence enabled: last known rider position shown when network drops
- [ ] Automatic reconnection when network is restored

**Phase:** MVP

---

### T5.2 — FCM Push Notifications

**Description:** Firebase Cloud Messaging for transactional push notifications. Sends order status updates, delivery updates, and chat message alerts. No marketing pushes in MVP.

**Dependencies:** T5.1 (Firebase project setup), T1.1 (user service for device tokens).

**Acceptance Criteria:**
- [ ] FCM configured in Firebase project
- [ ] Customer app registers for push notifications on launch, sends device token to backend
- [ ] Backend stores device tokens per user in PostgreSQL
- [ ] Push notifications sent for: order confirmed, order preparing, rider assigned, order on the way, order delivered, order cancelled
- [ ] Push data payload includes: `type`, `order_id`, `status`, `deep_link` (e.g., `talabat://orders/{id}/tracking`)
- [ ] Notification channel: single "Transactional" channel for all MVP notifications
- [ ] Tapping notification opens the order tracking screen via deep link
- [ ] Push sent via Firebase Admin SDK from backend (no third-party service)
- [ ] FCM message format matches the full architecture's data structure for future compatibility

**Phase:** MVP

---

### T5.3 — Rider-Customer Chat (Simplified)

**Description:** Basic text chat between rider and customer using Firebase RTDB. Messages are stored in `order_tracking/{order_id}/chat/`. This is a lightweight implementation compared to the full architecture which includes image sharing, predefined messages, and support agent integration.

**Dependencies:** T5.1 (Firebase RTDB).

**Acceptance Criteria:**
- [ ] Chat screen accessible from order tracking screen (chat icon button)
- [ ] Message list with sender name, text, timestamp
- [ ] Text input field with send button
- [ ] Messages written to RTDB: `{sender_type: "customer"|"rider", message_text, sent_at}`
- [ ] Customer app uses `ChildEventListener` on chat path for real-time message delivery
- [ ] Push notification sent when new chat message arrives (if app is backgrounded)
- [ ] No image sharing, no predefined messages, no support agent in MVP
- [ ] Chat history available only while order is active (no persistent chat history in MVP)

**Phase:** MVP

---

### T5.4 — Real-Time Order Status Flow

**Description:** The pipeline that updates order status from the backend to the customer app in real-time. When the vendor accepts an order or the rider updates delivery status, the backend writes to Firebase RTDB and sends a push notification simultaneously.

**Dependencies:** T1.3 (order service), T5.1 (Firebase RTDB), T5.2 (FCM push).

**Acceptance Criteria:**
- [ ] Order status change flow:
  1. Backend updates order status in PostgreSQL
  2. Backend writes new status to Firebase RTDB: `order_tracking/{order_id}/status`
  3. Backend sends FCM push notification with status and deep link
  4. Customer app: RTDB listener fires → UI updates immediately
  5. Customer app: push notification displayed (if app is backgrounded)
- [ ] Rider location update flow:
  1. Rider app sends GPS coordinates to Firebase RTDB every 5–10 seconds
  2. Customer app: RTDB listener fires → map marker moves
  3. Backend: ETA recalculated every 30 seconds (simple distance-based formula for MVP)
  4. If ETA change > 5 minutes → push notification sent
- [ ] Status values: ordered, preparing, delivering, delivered, cancelled
- [ ] Each status transition logged with timestamp in order timeline

**Phase:** MVP

---

### T5.5 — Network Resilience

**Description:** Handles offline scenarios gracefully. When the customer's device loses network during order tracking, the app shows the last known state and automatically recovers when connectivity returns.

**Dependencies:** T5.1 (Firebase RTDB with offline persistence), T2.12 (network layer).

**Acceptance Criteria:**
- [ ] Firebase RTDB offline persistence enabled: last 24 hours of tracking data cached on device
- [ ] When network lost during tracking: show last known rider position with "Reconnecting..." indicator
- [ ] When network restored: Firebase auto-syncs to latest state, rider position updates
- [ ] When network lost during chat: messages queued locally and sent on reconnection
- [ ] Connectivity monitoring via `connectivity_plus` plugin: show/hide offline banner
- [ ] Cart operations work offline using local SQLite cache; synced when online

**Phase:** MVP

---

## Phase 2 Real-Time Items (Not for MVP)

### T5.P2.1 — HMS Push Kit Support
**Description:** Add Huawei Mobile Services push for HMS-only devices. Requires dual FCM/HMS push logic in backend notification service. Currently FCM only.
**Phase:** Phase 2

### T5.P2.2 — Live Activity (Android Foreground Service)
**Description:** Persistent order tracking notification as a foreground service on Android. Shows rider status and ETA in the notification bar without opening the app. Requires native Android code.
**Phase:** Phase 2

### T5.P2.3 — iOS Live Activity (Dynamic Island)
**Description:** Apple ActivityKit integration for Dynamic Island and Lock Screen order tracking widgets on iOS 16.1+.
**Phase:** Phase 2

### T5.P2.4 — WebSocket Migration
**Description:** Replace Firebase RTDB with a custom WebSocket server for order tracking. Reduces cost at scale and provides greater control. OkHttp WebSocket is already bundled in the Talabat codebase.
**Phase:** Phase 2

### T5.P2.5 — SSE for Menu Availability
**Description:** Server-Sent Events for real-time menu item availability changes while the customer is browsing a vendor's menu. Prevents ordering out-of-stock items.
**Phase:** Phase 2

### T5.P2.6 — Advanced Chat Features
**Description:** Image sharing in chat, predefined quick messages ("I'm at the gate", "Come to lobby"), support agent integration, and chat history persistence.
**Phase:** Phase 2

### T5.P2.7 — Rider ETA Engine (ML-Based)
**Description:** Replace simple distance-based ETA with ML model combining historical delivery data, real-time traffic, weather, and order complexity. Currently ETA is straight-line distance / average speed.
**Phase:** Phase 2

### T5.P2.8 — Kafka Event Streaming
**Description:** Replace direct function calls for order status changes with Kafka event bus. Enables async processing, event replay, and decoupled services.
**Phase:** Phase 2
