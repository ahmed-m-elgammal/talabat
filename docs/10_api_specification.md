# 10 — API Specification

## 1. Overview

This document specifies the RESTful API interfaces exposed by Talabat's backend services to the Flutter mobile client. The API follows REST conventions with JSON request/response payloads, JWT Bearer authentication, and standard HTTP status codes. All endpoints are versioned (`/v1/`, `/v2/`) and accessed through the API Gateway which handles rate limiting, authentication, and request routing.

The API is designed with a **mobile-first** approach, optimizing for minimal payload sizes, efficient caching (ETags, conditional requests), and offline resilience. Where applicable, the specification references feature flags that gate API behavior, reflecting Talabat's extensive A/B testing infrastructure.

---

## 2. API Conventions

### 2.1 Base URLs

| Environment | Base URL |
|-------------|----------|
| Production (UAE) | `https://api.talabat.ae/v1` |
| Production (Kuwait) | `https://api.talabat.com.kw/v1` |
| Production (Egypt) | `https://api.talabat.eg/v1` |
| Production (Bahrain) | `https://api.talabat.bh/v1` |
| Production (Oman) | `https://api.talabat.om/v1` |
| Production (Qatar) | `https://api.talabat.qa/v1` |
| Production (Saudi) | `https://api.talabat.sa/v1` |
| Production (Jordan) | `https://api.talabat.jo/v1` |
| Production (Iraq) | `https://api.talabat.iq/v1` |
| Staging | `https://api.stg.talabat.net/v1` |

### 2.2 Authentication

```
Authorization: Bearer <access_token>
X-Device-ID: <device_uuid>
X-Country-Code: AE
X-App-Version: 13.58.1
X-Platform: android
X-Correlation-ID: <uuid>
```

### 2.3 Common Headers

| Header | Direction | Required | Description |
|--------|-----------|----------|-------------|
| `Authorization` | Request | Yes (except public endpoints) | Bearer JWT token |
| `X-Device-ID` | Request | Yes | Unique device identifier |
| `X-Country-Code` | Request | Yes | ISO country code |
| `X-App-Version` | Request | Yes | App version string |
| `X-Platform` | Request | Yes | `android` or `ios` |
| `X-Correlation-ID` | Request | No | Distributed tracing ID |
| `If-None-Match` | Request | No | ETag for conditional GET |
| `ETag` | Response | Conditional | Content hash for caching |
| `X-Request-ID` | Response | Always | Unique request identifier |
| `X-RateLimit-Remaining` | Response | Always | Remaining rate limit quota |

### 2.4 Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request body contains invalid fields",
    "details": [
      {
        "field": "phone_number",
        "message": "Invalid phone number format"
      }
    ],
    "request_id": "uuid",
    "timestamp": "2026-01-15T10:30:00Z"
  }
}
```

### 2.5 Standard Error Codes

| HTTP Status | Error Code | Description |
|-------------|-----------|-------------|
| 400 | `VALIDATION_ERROR` | Invalid request body or parameters |
| 401 | `UNAUTHORIZED` | Missing or invalid authentication token |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `NOT_FOUND` | Resource not found |
| 409 | `CONFLICT` | Resource state conflict (e.g., item out of stock) |
| 422 | `UNPROCESSABLE_ENTITY` | Business rule violation |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Unexpected server error |
| 503 | `SERVICE_UNAVAILABLE` | Service temporarily unavailable |

---

## 3. Authentication APIs

### 3.1 Send OTP

```
POST /auth/otp/send
```

**Request:**
```json
{
  "phone_number": "+971501234567",
  "country_code": "AE",
  "recaptcha_token": "string",
  "device_id": "uuid"
}
```

**Response (200):**
```json
{
  "request_id": "uuid",
  "expires_at": "2026-01-15T10:35:00Z",
  "is_new_user": false
}
```

**Error (429):**
```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many OTP requests. Try again in 5 minutes.",
    "retry_after_seconds": 300
  }
}
```

### 3.2 Verify OTP

```
POST /auth/otp/verify
```

**Request:**
```json
{
  "phone_number": "+971501234567",
  "otp_code": "123456",
  "request_id": "uuid",
  "device_id": "uuid"
}
```

**Response (200):**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2g...",
  "expires_in": 3600,
  "user": {
    "id": "uuid",
    "first_name": "Ahmed",
    "last_name": "Al-Rashid",
    "phone_number": "+971501234567",
    "email": "ahmed@example.com",
    "country_code": "AE",
    "is_pro_subscriber": true,
    "talabat_reference_number": "TB-AE-123456"
  }
}
```

### 3.3 Social Login

```
POST /auth/social/login
```

**Request:**
```json
{
  "provider": "google|apple|facebook",
  "id_token": "provider_id_token",
  "device_id": "uuid",
  "country_code": "AE"
}
```

**Response (200):**
```json
{
  "access_token": "...",
  "refresh_token": "...",
  "expires_in": 3600,
  "is_new_user": true,
  "linked_account_exists": false
}
```

### 3.4 Refresh Token

```
POST /auth/token/refresh
```

**Request:**
```json
{
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2g..."
}
```

**Response (200):**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "refresh_token": "bmV3IHJlZnJlc2ggdG9rZW4...",
  "expires_in": 3600
}
```

---

## 4. User APIs

### 4.1 Get Profile

```
GET /users/me
```

**Response (200):**
```json
{
  "id": "uuid",
  "first_name": "Ahmed",
  "last_name": "Al-Rashid",
  "email": "ahmed@example.com",
  "phone_number": "+971501234567",
  "date_of_birth": "1990-05-15",
  "gender": "male",
  "country_code": "AE",
  "talabat_reference_number": "TB-AE-123456",
  "newsletter_subscribed": true,
  "pro_subscription": {
    "status": "active",
    "plan_type": "pro",
    "expiry_date": "2026-02-15"
  },
  "wallet_balance": 150.00,
  "reward_points": 2500,
  "created_at": "2023-01-15T10:30:00Z"
}
```

### 4.2 Update Profile

```
PATCH /users/me
```

**Request:**
```json
{
  "first_name": "Ahmed",
  "last_name": "Al-Rashid",
  "email": "ahmed@example.com",
  "date_of_birth": "1990-05-15",
  "gender": "male"
}
```

### 4.3 Get Addresses

```
GET /users/me/addresses
```

**Response (200):**
```json
{
  "addresses": [
    {
      "id": "uuid",
      "label": "home",
      "area_name": "Dubai Marina",
      "latitude": 25.0805,
      "longitude": 55.1403,
      "building": "Tower A",
      "floor": "15",
      "apartment": "1502",
      "delivery_instructions": "Ring buzzer 1502",
      "is_default": true,
      "geohash": "thrzg2w"
    }
  ]
}
```

### 4.4 Create Address

```
POST /users/me/addresses
```

**Request:**
```json
{
  "label": "work",
  "area_name": "DIFC",
  "latitude": 25.2117,
  "longitude": 55.2708,
  "building": "Gate Village",
  "floor": "3",
  "apartment": "305",
  "delivery_instructions": "Security desk entrance",
  "is_default": false
}
```

---

## 5. Vendor APIs

### 5.1 Get Vendors (Listing)

```
GET /vendors?latitude={lat}&longitude={lng}&vertical={type}&page={n}&limit={n}
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `latitude` | float | Yes | Delivery location latitude |
| `longitude` | float | Yes | Delivery location longitude |
| `vertical` | string | Yes | `food`, `grocery`, `pharmacy`, `flowers`, `speciality` |
| `page` | int | No | Page number (default 1) |
| `limit` | int | No | Items per page (default 20, max 50) |
| `cuisine` | string[] | No | Filter by cuisine type |
| `sort` | string | No | `recommended`, `delivery_time`, `rating`, `distance`, `min_order` |
| `is_pro_free_delivery` | boolean | No | Filter Pro-eligible vendors |
| `is_open` | boolean | No | Filter by open status |
| `offer_available` | boolean | No | Filter vendors with active offers |

**Response (200):**
```json
{
  "vendors": [
    {
      "id": "uuid",
      "name": "Al Safadi",
      "name_ar": "الصفدي",
      "vertical_type": "food",
      "logo_url": "https://cdn.talabat.ae/logos/...",
      "cover_url": "https://cdn.talabat.ae/covers/...",
      "cuisine_types": ["Lebanese", "Middle Eastern"],
      "rating": {
        "average": 4.5,
        "count": 12580
      },
      "delivery_time": {
        "min": 30,
        "max": 45
      },
      "delivery_fee": 5.00,
      "minimum_order_value": 20.00,
      "is_open": true,
      "is_busy": false,
      "is_pro_free_delivery": true,
      "is_tgo_available": true,
      "badges": ["high_rated", "always_on_time"],
      "sponsored": false,
      "offers": [
        {
          "type": "percentage_discount",
          "value": 20,
          "min_order": 50,
          "valid_until": "2026-01-20T23:59:59Z"
        }
      ],
      "distance_km": 2.5
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 156,
    "has_more": true
  }
}
```

### 5.2 Get Vendor Menu

```
GET /vendors/{vendor_id}/menu
If-None-Match: "etag_hash"
```

**Response (200):**
```json
{
  "vendor": {
    "id": "uuid",
    "name": "Al Safadi",
    "legal_name": "Al Safadi Restaurant LLC",
    "vertical_type": "food",
    "is_open": true,
    "delivery_time": { "min": 30, "max": 45 },
    "minimum_order_value": 20.00,
    "delivery_fee": 5.00,
    "inclusive_vat": true,
    "rating": { "average": 4.5, "count": 12580 },
    "reviews_summary": {
      "total": 12580,
      "rating_distribution": { "5": 7500, "4": 3000, "3": 1500, "2": 500, "1": 80 }
    },
    "info": {
      "address": "Dubai Marina Walk",
      "hours": "10:00 AM - 12:00 AM",
      "phone": "+97141234567"
    },
    "pro_info": {
      "is_pro_eligible": true,
      "free_delivery": true,
      "pro_discount": 0
    }
  },
  "categories": [
    {
      "id": "uuid",
      "name": "Starters",
      "name_ar": "مقبلات",
      "display_order": 1,
      "is_available": true,
      "items": [
        {
          "id": "uuid",
          "name": "Hummus",
          "name_ar": "حمص",
          "description": "Creamy chickpea dip with tahini",
          "description_ar": "حمص كريمي مع طحينة",
          "base_price": 18.00,
          "price_on_selection": false,
          "is_available": true,
          "image_url": "https://cdn.talabat.ae/items/...",
          "preparation_time_minutes": 5,
          "is_age_restricted": false,
          "options": [
            {
              "id": "uuid",
              "name": "Size",
              "name_ar": "الحجم",
              "is_required": true,
              "min_selections": 1,
              "max_selections": 1,
              "display_order": 1,
              "choices": [
                {
                  "id": "uuid",
                  "name": "Regular",
                  "name_ar": "عادي",
                  "price_modifier": 0,
                  "is_default": true,
                  "is_available": true,
                  "calories": 200
                },
                {
                  "id": "uuid",
                  "name": "Large",
                  "name_ar": "كبير",
                  "price_modifier": 8.00,
                  "is_default": false,
                  "is_available": true,
                  "calories": 350
                }
              ]
            }
          ]
        }
      ]
    }
  ],
  "etag": "menu_hash_abc123"
}
```

---

## 6. Order APIs

### 6.1 Create Order

```
POST /orders
```

**Request:**
```json
{
  "vendor_id": "uuid",
  "delivery_address_id": "uuid",
  "delivery_type": "delivery",
  "items": [
    {
      "menu_item_id": "uuid",
      "quantity": 2,
      "selected_choices": {
        "option_id_1": ["choice_id_1"],
        "option_id_2": ["choice_id_1", "choice_id_2"]
      },
      "special_instructions": "Extra spicy"
    }
  ],
  "voucher_code": "SAVE20",
  "payment_method": {
    "type": "card",
    "token_id": "uuid"
  },
  "rider_tip": 5.00,
  "scheduled_delivery_time": null,
  "is_contactless": true,
  "is_tgo": false
}
```

**Response (201):**
```json
{
  "order_id": "uuid",
  "order_number": "TB-AE-20260115-12345",
  "status": "ordered",
  "subtotal": 85.00,
  "delivery_fee": 5.00,
  "service_fee": 4.25,
  "municipality_tax": 0.00,
  "tourist_tax": 0.00,
  "discount": 0.00,
  "voucher_discount": 17.00,
  "rider_tip": 5.00,
  "total": 82.25,
  "currency": "AED",
  "estimated_delivery_time": {
    "min": 30,
    "max": 45
  },
  "payment_status": "authorized",
  "tracking": {
    "firebase_path": "order_tracking/uuid",
    "live_activity_supported": true
  },
  "created_at": "2026-01-15T10:30:00Z"
}
```

### 6.2 Get Order

```
GET /orders/{order_id}
```

**Response (200):**
```json
{
  "order_id": "uuid",
  "order_number": "TB-AE-20260115-12345",
  "status": "delivering",
  "vendor": {
    "id": "uuid",
    "name": "Al Safadi",
    "logo_url": "..."
  },
  "items": [
    {
      "id": "uuid",
      "name": "Hummus",
      "quantity": 2,
      "unit_price": 18.00,
      "total_price": 36.00,
      "selected_options_display": "Large",
      "status": "confirmed",
      "special_instructions": "Extra spicy"
    }
  ],
  "pricing": {
    "subtotal": 85.00,
    "delivery_fee": 5.00,
    "service_fee": 4.25,
    "municipality_tax": 0.00,
    "tourist_tax": 0.00,
    "discount": 0.00,
    "voucher_discount": 17.00,
    "rider_tip": 5.00,
    "total": 82.25,
    "currency": "AED"
  },
  "delivery": {
    "type": "delivery",
    "address": "Tower A, Floor 15, Apt 1502, Dubai Marina",
    "is_contactless": true,
    "is_tgo": false,
    "rider": {
      "id": "uuid",
      "name": "Muhammad",
      "phone": "+971509876543",
      "vehicle_type": "motorcycle",
      "rating": 4.8
    },
    "estimated_arrival": "2026-01-15T11:00:00Z",
    "tracking_enabled": true
  },
  "payment": {
    "method": "card",
    "last_four": "4242",
    "brand": "visa",
    "status": "captured"
  },
  "modifications": [],
  "timeline": [
    { "status": "ordered", "timestamp": "2026-01-15T10:30:00Z" },
    { "status": "preparing", "timestamp": "2026-01-15T10:32:00Z" },
    { "status": "delivering", "timestamp": "2026-01-15T10:50:00Z" }
  ],
  "created_at": "2026-01-15T10:30:00Z",
  "updated_at": "2026-01-15T10:50:00Z"
}
```

### 6.3 Get Order History

```
GET /orders?page={n}&limit={n}&status={status}
```

### 6.4 Cancel Order

```
POST /orders/{order_id}/cancel
```

**Request:**
```json
{
  "reason": "changed_mind",
  "comment": "Ordered by mistake"
}
```

### 6.5 Get Order Invoice

```
GET /orders/{order_id}/invoice
```
(Feature flag: `ff_ordering_show_order_invoice`)

---

## 7. Cart APIs

### 7.1 Get Cart

```
GET /cart
```

### 7.2 Add Item to Cart

```
POST /cart/items
```

**Request:**
```json
{
  "menu_item_id": "uuid",
  "vendor_id": "uuid",
  "quantity": 1,
  "selected_choices": {
    "option_id": ["choice_id_1"]
  },
  "special_instructions": ""
}
```

### 7.3 Update Cart Item

```
PUT /cart/items/{item_id}
```

### 7.4 Remove Cart Item

```
DELETE /cart/items/{item_id}
```

### 7.5 Apply Voucher

```
POST /cart/voucher
```

**Request:**
```json
{
  "voucher_code": "SAVE20"
}
```

**Error (422) — BIN Voucher:**
```json
{
  "error": {
    "code": "BIN_VOUCHER_INVALID",
    "message": "This voucher requires a specific card type",
    "title": "BIN voucher not applicable",
    "primary_action": "Change payment method",
    "secondary_action": "Remove voucher"
  }
}
```

---

## 8. Payment APIs

### 8.1 Get Payment Methods

```
GET /payments/methods
```

**Response (200):**
```json
{
  "methods": [
    {
      "id": "uuid",
      "type": "card",
      "last_four": "4242",
      "brand": "visa",
      "expiry_month": 12,
      "expiry_year": 2028,
      "is_default": true,
      "provider": "checkout_com"
    },
    {
      "id": "uuid",
      "type": "apple_pay",
      "is_default": false
    },
    {
      "id": "uuid",
      "type": "wallet",
      "balance": 150.00,
      "currency": "AED",
      "is_default": false
    },
    {
      "id": "uuid",
      "type": "bnpl",
      "available_limit": 2000.00,
      "currency": "AED",
      "is_default": false
    }
  ],
  "available_methods": ["card", "apple_pay", "google_pay", "wallet", "bnpl", "stc_pay", "benefit_pay", "zaincash", "cash"]
}
```

### 8.2 Tokenize Card (Checkout.com)

```
POST /payments/tokenize/card
```

**Request:**
```json
{
  "card_number": "4242424242424242",
  "expiry_month": 12,
  "expiry_year": 2028,
  "cvv": "123",
  "cardholder_name": "Ahmed Al-Rashid"
}
```

**Response (200):**
```json
{
  "token_id": "uuid",
  "token_type": "card",
  "last_four": "4242",
  "brand": "visa",
  "expiry_month": 12,
  "expiry_year": 2028,
  "provider": "checkout_com",
  "three_ds_required": true,
  "three_ds_url": "https://api.checkout.com/tokens/..."
}
```

### 8.3 Initiate Payment (HyperPay)

```
POST /payments/checkout
```

**Request:**
```json
{
  "order_id": "uuid",
  "payment_method_id": "uuid",
  "amount": 82.25,
  "currency": "AED",
  "payment_brand": "VISA",
  "shopper_result_url": "talabat://payment-result"
}
```

---

## 9. Search APIs

### 9.1 Search

```
GET /search?q={query}&vertical={type}&latitude={lat}&longitude={lng}&page={n}&limit={n}
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `q` | string | Yes | Search query |
| `vertical` | string | No | `food`, `grocery`, `pharmacy`, `flowers`, `all` |
| `latitude` | float | Yes | User latitude |
| `longitude` | float | Yes | User longitude |
| `sort` | string | No | `relevance`, `delivery_time`, `rating`, `distance` |
| `cuisine` | string[] | No | Cuisine filter |
| `price_range` | string | No | `budget`, `moderate`, `premium` |
| `is_open` | boolean | No | Filter by open status |
| `is_pro_free_delivery` | boolean | No | Pro free delivery filter |
| `page` | int | No | Page number |
| `limit` | int | No | Results per page |

**Response (200):**
```json
{
  "query": "burger",
  "vertical": "food",
  "sections": {
    "vendors": {
      "total": 45,
      "results": [
        {
          "id": "uuid",
          "name": "Burger King",
          "cuisine_types": ["Burger", "Fast Food"],
          "rating": { "average": 4.2, "count": 5600 },
          "delivery_time": { "min": 25, "max": 35 },
          "delivery_fee": 3.00,
          "is_open": true,
          "is_pro_free_delivery": true,
          "sponsored": true,
          "sponsored_label": "Ad",
          "logo_url": "..."
        }
      ]
    },
    "items": {
      "total": 120,
      "results": [
        {
          "id": "uuid",
          "name": "Whopper Meal",
          "vendor_name": "Burger King",
          "price": 35.00,
          "image_url": "..."
        }
      ]
    }
  },
  "suggested_queries": ["burger king", "burger meal", "smash burger"],
  "cached": false
}
```

### 9.2 Autocomplete

```
GET /search/autocomplete?q={prefix}&vertical={type}
```

### 9.3 Multi-Search (Q-Commerce)

```
POST /search/multi
```

**Request:**
```json
{
  "queries": ["milk", "bread", "eggs"],
  "latitude": 25.0805,
  "longitude": 55.1403
}
```

(Feature flag: `exp_qcommerce_multi_search`)

---

## 10. Wallet APIs

### 10.1 Get Wallet Balance

```
GET /wallet
```

### 10.2 Top Up Wallet

```
POST /wallet/topup
```

**Request:**
```json
{
  "amount": 100.00,
  "payment_method_id": "uuid"
}
```

### 10.3 Get Transaction History

```
GET /wallet/transactions?page={n}&limit={n}
```

---

## 11. Subscription APIs

### 11.1 Get Subscription Plans

```
GET /subscriptions/plans
```

### 11.2 Subscribe

```
POST /subscriptions
```

**Request:**
```json
{
  "plan_type": "pro",
  "payment_method_id": "uuid",
  "family_members": ["+971501111111", "+971502222222"]
}
```

### 11.3 Cancel Subscription

```
POST /subscriptions/cancel
```

**Request:**
```json
{
  "reason": "too_expensive",
  "comment": "Saving money"
}
```

---

## 12. BNPL APIs

### 12.1 Get BNPL Dashboard

```
GET /bnpl/dashboard
```

### 12.2 Pay Installment

```
POST /bnpl/pay
```

### 12.3 Rewind (Convert to PostPaid)

```
POST /bnpl/rewind
```

**Request:**
```json
{
  "order_id": "uuid",
  "payment_method_id": "uuid"
}
```

---

## 13. Notification APIs

### 13.1 Get Notification Preferences

```
GET /notifications/preferences
```

### 13.2 Update Notification Preferences

```
PUT /notifications/preferences
```

---

## 14. Rewards APIs

### 14.1 Get Points Balance

```
GET /rewards/points
```

### 14.2 Get Burn Options

```
GET /rewards/burn-options
```

### 14.3 Redeem Points

```
POST /rewards/redeem
```

**Request:**
```json
{
  "points_amount": 500,
  "burn_option": "free_delivery",
  "order_id": "uuid"
}
```

---

## 15. Rate Limiting

| Endpoint Category | Rate Limit | Window |
|-------------------|-----------|--------|
| Authentication | 10 requests | 1 minute |
| OTP requests | 3 requests | 5 minutes |
| Search | 30 requests | 1 minute |
| Vendor listing | 30 requests | 1 minute |
| Order creation | 5 requests | 1 minute |
| Cart operations | 30 requests | 1 minute |
| Payment operations | 10 requests | 1 minute |
| General API | 100 requests | 1 minute |

Rate limit headers are included in every response:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1700000060
```
