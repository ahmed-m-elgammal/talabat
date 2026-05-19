# 08 — Notification System MVP

## Overview

The MVP notification system delivers **transactional push notifications only** for order status updates, delivery tracking, and chat messages. The full Talabat architecture (see `docs/15_notification_system.md`) is a multi-channel, multi-provider infrastructure with Braze for marketing/lifecycle campaigns, FCM + HMS dual push delivery, 12 Android notification channels, in-app message factory, smart suppression rules, and live activity foreground service — serving transactional, engagement, and operational objectives across 9 countries.

The MVP strips this down to **FCM-only push** for transactional notifications (order confirmed, preparing, on the way, delivered, cancelled) and chat messages, with a single notification channel and simple in-app toast/banner for foreground display. Braze, HMS, marketing campaigns, in-app messages, smart suppression, and notification preferences are all deferred to Phase 2.

---

## MVP Tasks

### T8.1 — FCM Project Setup & Device Token Management

**Description:** Sets up Firebase Cloud Messaging in the project and implements device token registration flow. The mobile app requests push notification permissions on first launch, obtains an FCM device token, and sends it to the backend. The backend stores the token per user (one token per device, users can have multiple devices). Tokens are refreshed when the app updates or FCM invalidates them. This is the foundation for all push notification delivery in MVP.

**Dependencies:** T1.1 (user service for authentication), T1.7 (database schema for token storage).

**Acceptance Criteria:**
- [ ] Firebase project has Cloud Messaging enabled with server key configured
- [ ] Mobile app requests notification permission on first launch (iOS: `UNUserNotificationCenter.requestAuthorization`, Android: `POST_NOTIFICATIONS` runtime permission on Android 13+)
- [ ] On permission granted, app obtains FCM device token via `FirebaseMessaging.getInstance().getToken()`
- [ ] `POST /v1/users/me/devices` stores device token with: `token`, `platform` (ios/android), `app_version`, `created_at`
- [ ] `DELETE /v1/users/me/devices/{token}` removes a device token (called when user logs out)
- [ ] Backend handles token refresh: app sends updated token via same POST endpoint (upsert by token value)
- [ ] Database table `user_devices`: `{id, user_id, fcm_token, platform, app_version, created_at, updated_at}`
- [ ] If user denies notification permission, app continues to function without push; in-app banners still work
- [ ] Token cleanup: cron job deletes tokens that have been invalid for > 7 days (FCM returns `UNREGISTERED`)

**Phase:** MVP

---

### T8.2 — Transactional Push Notification Service

**Description:** Implements the backend service that sends transactional push notifications for order lifecycle events. When an order status changes (confirmed, preparing, on the way, delivered, cancelled), the backend sends an FCM push to the customer's registered devices. The notification includes a data payload for deep linking and a visible title/body for the system notification tray. This service uses Firebase Admin SDK directly (no Braze or third-party push service in MVP).

**Dependencies:** T8.1 (FCM setup and device tokens), T1.3 (order service for status change events).

**Acceptance Criteria:**
- [ ] Push notifications sent for these order events:
  - Order confirmed: "Your order has been confirmed!" → deep link: `talabat://orders/{id}/tracking`
  - Order preparing: "The restaurant is preparing your order" → deep link: `talabat://orders/{id}/tracking`
  - Order on the way: "Your rider is on the way!" → deep link: `talabat://orders/{id}/tracking`
  - Order delivered: "Your order has been delivered. Enjoy!" → deep link: `talabat://orders/{id}`
  - Order cancelled: "Your order has been cancelled" → deep link: `talabat://orders/{id}`
- [ ] Push notifications sent for chat events:
  - New chat message: "Rider: {message_preview}" → deep link: `talabat://orders/{id}/chat`
- [ ] FCM data payload format: `{type: "order_status"|"chat_message", order_id, status, deep_link}`
- [ ] FCM notification payload: `{title, body, sound: "default"}`
- [ ] Push sent to ALL registered devices for the user (multi-device support)
- [ ] Failed sends logged: token invalid → mark for cleanup; rate limited → retry with exponential backoff (3 attempts)
- [ ] Notification content stored in `notification_log` table: `{id, user_id, type, order_id, title, body, sent_at, delivery_status}`
- [ ] Push sent within 2 seconds of the triggering event
- [ ] Uses Firebase Admin SDK `MulticastMessage` for efficient multi-device delivery

**Phase:** MVP

---

### T8.3 — Notification Channel Setup (Android)

**Description:** Creates the Android notification channels required for MVP. For the MVP, only two channels are needed: "Transactional" (for order status updates) and "Chat" (for rider-customer messages). This is a minimal subset of the full Talabat app's 12 channels. Channel configuration follows Android best practices with appropriate importance levels and sound settings.

**Dependencies:** T2.1 (Flutter app shell with native Android integration).

**Acceptance Criteria:**
- [ ] `Transactional` channel created in `Application.onCreate()`:
  - ID: `transactional`
  - Importance: DEFAULT (128) — visible but not heads-up for every order update
  - Sound: default
  - Description: "Order status and delivery updates"
- [ ] `Chat` channel created in `Application.onCreate()`:
  - ID: `chat`
  - Importance: HIGH (224) — heads-up notification for real-time chat messages
  - Sound: default
  - Description: "Messages from your rider"
- [ ] Both channels registered via `NotificationManagerCompat.createNotificationChannel()`
- [ ] Channel IDs used consistently when building notifications (no hardcoded channel strings)
- [ ] Existing users who upgrade: channels created silently without re-prompting for permission

**Phase:** MVP

---

### T8.4 — Push Notification Handler (Foreground & Background)

**Description:** Implements the client-side push notification handling logic for both foreground and background states. When the app is in the foreground, notifications are displayed as an in-app banner (not system notification). When the app is in the background, the system notification tray shows the notification. Tapping a notification navigates to the appropriate screen via deep link. This replaces the full architecture's `TalabatFirebaseMessagingService` (which extends `BrazeFirebaseMessagingService`) with a simpler FCM-only implementation.

**Dependencies:** T8.1 (FCM setup), T8.3 (notification channels), T2.6 (order tracking screen for deep link target).

**Acceptance Criteria:**
- [ ] `FirebaseMessagingService` subclass registered in AndroidManifest.xml
- [ ] `onMessageReceived()` handles both foreground and background messages:
  - **Foreground**: show in-app toast/banner at top of screen with title, body, and auto-dismiss after 5 seconds; tapping navigates via deep link
  - **Background**: FCM displays system notification automatically (via `notification` payload); tapping opens app and navigates via deep link
- [ ] Deep link routing: parse `deep_link` from data payload and navigate to correct screen:
  - `/orders/{id}/tracking` → OrderTrackingScreen
  - `/orders/{id}/chat` → ChatScreen
  - `/orders/{id}` → OrderDetailScreen
  - Unknown deep link → HomeScreen
- [ ] Notification tapped while app is killed: app launches, auth check runs, then navigates to deep link target
- [ ] Multiple notifications for same order: latest notification replaces previous (grouped by order_id using `tag` parameter)
- [ ] iOS: `UNUserNotificationCenter.delegate` handles foreground display with `willPresent` callback

**Phase:** MVP

---

### T8.5 — In-App Notification Banner

**Description:** Implements a lightweight in-app notification banner that appears at the top of the screen when a push notification arrives while the app is in the foreground. The banner shows the notification title and body, auto-dismisses after 5 seconds, and can be tapped to navigate to the relevant screen. This provides a non-intrusive way to inform the user of order updates without forcing them to check the notification tray. The full architecture uses Braze's in-app message factory with full-screen modals, slide-ups, and HTML5 messages; the MVP uses a simple custom banner.

**Dependencies:** T8.4 (push handler that triggers the banner).

**Acceptance Criteria:**
- [ ] Banner appears at the top of the screen, sliding down with a 200ms animation
- [ ] Banner content: icon (order/chat), title text, body text (max 2 lines, truncated with ellipsis)
- [ ] Banner auto-dismisses after 5 seconds with a 200ms slide-up animation
- [ ] Banner is tappable: navigates to the deep link target screen
- [ ] Banner can be dismissed by swiping up
- [ ] Multiple notifications in quick succession: banner is replaced (not stacked) — latest wins
- [ ] Banner does not block interaction with the rest of the screen (non-modal)
- [ ] Banner respects safe area on iOS (not hidden by notch/dynamic island)
- [ ] Banner styled consistently with app theme: white background, subtle shadow, brand accent color for icon

**Phase:** MVP

---

### T8.6 — Order Status Notification Content (Localization-Ready)

**Description:** Defines the notification title and body content for each order status event, structured for future localization. For MVP, content is in English only, but the content is stored as string resources (not hardcoded) to make adding Arabic and other languages trivial in Phase 2. Includes support for dynamic content interpolation (vendor name, ETA, rider name).

**Dependencies:** T8.2 (push notification service that sends these messages).

**Acceptance Criteria:**
- [ ] Notification content defined in a centralized configuration (not scattered in business logic):
  ```
  order.confirmed.title = "Order Confirmed"
  order.confirmed.body = "Your order from {vendor_name} has been confirmed!"
  order.preparing.title = "Preparing Your Order"
  order.preparing.body = "{vendor_name} is preparing your order"
  order.delivering.title = "On the Way"
  order.delivering.body = "Your rider is on the way! ETA: {eta_minutes} min"
  order.delivered.title = "Delivered"
  order.delivered.body = "Your order from {vendor_name} has been delivered. Enjoy!"
  order.cancelled.title = "Order Cancelled"
  order.cancelled.body = "Your order from {vendor_name} has been cancelled"
  chat.message.title = "New Message"
  chat.message.body = "{sender_name}: {message_preview}"
  ```
- [ ] Dynamic placeholders (`{vendor_name}`, `{eta_minutes}`, `{sender_name}`) resolved at send time from order/rider data
- [ ] Message preview truncated to 50 characters with ellipsis
- [ ] Content stored as string resources / configuration file (not inline strings in code)
- [ ] All content values have sensible defaults if dynamic data is unavailable (e.g., "Your order" if vendor_name is null)
- [ ] Configuration is easily extensible for Arabic (right-to-left) in Phase 2

**Phase:** MVP

---

### T8.7 — Notification Logging & Delivery Tracking

**Description:** Implements server-side logging for all sent notifications and basic delivery tracking. Every notification attempt is logged with its delivery status. Provides a simple admin endpoint to view notification history for a user or order. This is the foundation for Phase 2 analytics (open rates, conversion rates) but MVP only tracks send/delivery status, not opens or clicks.

**Dependencies:** T8.2 (push notification service), T1.7 (database schema).

**Acceptance Criteria:**
- [ ] `notification_log` table: `{id, user_id, type, order_id, fcm_token, title, body, sent_at, delivery_status, error_message}`
- [ ] Delivery status values: `sent`, `delivered`, `failed`
- [ ] FCM response processed: successful sends → `delivered`, errors → `failed` with error message
- [ ] FCM error categorization: `UNREGISTERED` (token invalid), `QUOTA_EXCEEDED` (rate limited), `INTERNAL` (FCM error), `THIRD_PARTY` (other)
- [ ] `GET /v1/admin/users/{id}/notifications` returns notification history for a user (admin only)
- [ ] `GET /v1/admin/orders/{id}/notifications` returns notifications sent for an order (admin only)
- [ ] Notification logs retained for 90 days; older entries purged by weekly cron job
- [ ] Failed notification count alert: if failed > 10% in 1 hour, log a warning for investigation

**Phase:** MVP

---

## Phase 2 Notification Items (Not for MVP)

### T8.P2.1 — Braze Integration (Marketing & Lifecycle)
**Description:** Full Braze SDK integration for engagement campaigns: push notifications via `BrazeFirebaseMessagingService`, in-app messages with custom factory (`BrazeInAppMessageManager`), content cards for promotional home screen content, user attribute sync (first_name, pro_status, wallet_balance), and custom event tracking (order_placed, cart_abandoned, search_performed). Requires Braze API keys for Google Play and HMS endpoints.
**Phase:** Phase 2

### T8.P2.2 — HMS Push Kit Support
**Description:** Add Huawei Mobile Services push for HMS-only devices (Chinese market and some Middle Eastern devices). Backend notification routing: device has Google Play → FCM, HMS only → HMS Push Kit, both → FCM preferred. Dual token management in `user_devices` table.
**Phase:** Phase 2

### T8.P2.3 — Full Android Notification Channels (12 Channels)
**Description:** Expand from 2 MVP channels to 12 channels matching the full architecture: prospect, lifecycle_engagement, churning_winback, adhoc_restaurant_deals, adhoc_marketing_vouchers, abandonments, app_feature_updates, transactional, talabatgo, brand_updates, general, chat. Each with appropriate importance levels and user-toggleable settings.
**Phase:** Phase 2

### T8.P2.4 — Smart Suppression Rules
**Description:** Notification frequency management: daily cap (>5 marketing per 24h suppresses remaining), frequency cap (same campaign max 1x per 24h), sleep hours (queue 11PM–8AM for morning delivery), recent order suppression (no marketing <30min after order), uninstall detection (Braze bounce → stop sending), inactive user frequency reduction.
**Phase:** Phase 2

### T8.P2.5 — Live Activity (Android Foreground Service)
**Description:** Persistent order tracking notification as an Android foreground service. Shows rider icon, vendor name, ETA, and action buttons (Track, Chat, Call) in the notification bar. Created when order enters "delivering" state, updated with real-time ETA from Firebase RTDB, auto-dismissed on delivery, auto-expired after 2 hours.
**Phase:** Phase 2

### T8.P2.6 — iOS Live Activity (Dynamic Island)
**Description:** Apple ActivityKit integration for Dynamic Island and Lock Screen order tracking widgets on iOS 16.1+. Shows delivery progress with ETA, rider info, and tap-to-open tracking screen. Matches the Android Live Activity feature set.
**Phase:** Phase 2

### T8.P2.7 — In-App Message System (Braze-Powered)
**Description:** Full in-app messaging via Braze: full-screen modals for major announcements, slide-up banners for minor promotions, centered modal cards for targeted offers, and HTML5 custom messages (`braze-html-bridge.js` + `braze-html-in-app-message-bridge.js`). Custom `BrazeImageLoader` via Flutter plugin.
**Phase:** Phase 2

### T8.P2.8 — Notification Preferences Screen
**Description:** User-facing notification settings screen with per-channel toggle controls. Transactional channel cannot be disabled (regulatory requirement). All marketing channels individually toggleable. Persists preferences server-side and syncs to Braze for campaign targeting.
**Phase:** Phase 2

### T8.P2.9 — Notification Analytics (Perseus Integration)
**Description:** Full notification analytics pipeline: delivery rate (>95%), open rate (>15% marketing, >80% transactional), CTR (>30% marketing), conversion rate (>5% marketing), opt-out rate (<2%), bounce rate (<3%). Perseus events: `notification_received`, `notification_opened`, `notification_dismissed`, `notification_action_taken`, `in_app_message_shown`, `in_app_message_clicked`.
**Phase:** Phase 2

### T8.P2.10 — Compliance & Regulatory Framework
**Description:** Per-region opt-in requirements: UAE (TRA), Saudi Arabia (CITC with Arabic opt-out), Egypt (NTRA with local data processing), GDPR-equivalent (right to withdraw consent). Explicit consent checkbox during registration (`authentication.sign_up.offers.check.box.title`). Data residency: Braze US (`sdk.iad-01.braze.com`), AI chat disclaimer for OpenAI data transfer. 90-day notification log retention.
**Phase:** Phase 2
