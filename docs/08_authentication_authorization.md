# 08 — Authentication & Authorization

## 1. Overview

Talabat implements a multi-provider authentication system that supports mobile OTP, email/password, and social login (Google, Apple, Facebook) across 9 MENA countries. The authorization layer uses a role-based access control (RBAC) model with JWT-based session management, complemented by device fingerprinting and location-based fraud prevention. The auth system must balance security requirements (strong identity verification for financial operations) with low-friction onboarding (to maximize conversion in competitive markets).

The authentication flow is designed to be **progressive** — users can browse and explore without authentication, but must complete verification before placing orders or accessing payment features. This approach minimizes onboarding drop-off while maintaining security for sensitive operations.

---

## 2. Authentication Methods

### 2.1 Mobile OTP Authentication (Primary)

Mobile number verification via SMS OTP is the primary authentication method, reflecting the MENA region's preference for phone-based identity.

**Flow:**

```
User enters phone number
        │
        ▼
Client validates format (E.164, country-specific patterns)
        │
        ├── Validation error: "authentication.mobile.validation.error"
        │                  "authentication.phone.validation.error"
        ▼
API: POST /auth/otp/send
        │
        ├── Rate limit check (Redis: ratelimit:otp:{phone})
        │   └── Too many requests: "authentication.failed.too.many.requests.error"
        │
        ├── Phone exists?
        │   ├── YES → Login OTP flow
        │   │   Header: "authentication.login_otp.mobile.header"
        │   │   Title: "authentication.login_otp.mobile.title"
        │   │   Subtitle: "authentication.login_otp.mobile.subTitle"
        │   │
        │   └── NO → Registration flow
        │       Title: "authentication.sign_up.title"
        │       Fields: first_name, last_name, email (optional), DOB, password
        │
        ▼
SMS OTP sent (6-digit code, 5-minute expiry)
        │
        ▼
User enters OTP (SMS autofill via sms_autofill plugin)
        │
        ├── ReCAPTCHA Enterprise check (bot protection)
        ├── Incognia location verification (device/location consistency)
        │
        ├── OTP Valid?
        │   ├── YES → JWT tokens issued (access + refresh)
        │   │       "authentication.email_verification_otp.success.title"
        │   │       "authentication.email_verification_otp.success.subtitle"
        │   │
        │   └── NO → "authentication.login_otp.phone_exists.error"
        │            "authentication.login_otp.account.link.error"
```

**OTP Security Measures:**

| Measure | Implementation |
|---------|---------------|
| Rate limiting | Max 3 OTP requests per phone per 5 minutes |
| OTP expiry | 5 minutes |
| Attempt limit | Max 5 verification attempts per OTP |
| reCAPTCHA Enterprise | Bot detection on OTP request (`recaptcha_enterprise_flutter` plugin) |
| Incognia verification | Location consistency check (`incognia_flutter` plugin, App ID: `3df43253-f642-4402-86c0-4333fb93bf73`) |
| Shield Service | Device fingerprinting and fraud detection (`shield_service_plugin`) |
| VM detection | Flag `ff_is_vm_detection_enabled` blocks emulators |

### 2.2 Email/Password Authentication

Email-based authentication is a secondary method, supported for users who prefer email over phone:

```
Login Screen: "authentication.login.title"
        │
        ├── Email tab: "authentication.login.email.subtitle"
        │   ├── Email field: "authentication.email"
        │   │   Validation: "authentication.email.validation.error"
        │   │   Helper: "authentication.login.email.otp.helper.text"
        │   │
        │   ├── Password field: "authentication.login.password"
        │   │   Validation: "authentication.login.password.validation.error"
        │   │
        │   └── Forgot password: "authentication.login.forgot.password"
        │       "authentication.forgot_password.title"
        │       Dialog: "authentication.forgot_password.dialog.success.title"
        │
        ├── Mobile tab: "authentication.login.mobile.subtitle"
        │
        └── Create account: "authentication.login.create.an.account"
```

**Email OTP Login Flow (V2):**

The feature flag `exp_user_email_otp_login_flow_enabled_v2` enables a passwordless email login flow where:

1. User enters email address
2. OTP is sent to the email
3. User enters OTP instead of password
4. Helper text: `"authentication.login_otp.register.email.helper.text"`

**Password Reset Flow:**

```
"authentication.forgot_password.title"
        │
        ▼
Enter email or mobile
        │
        ├── Email: "authentication.forgot_password.dialog.email.error.title"
        ├── Mobile: "authentication.forgot_password.dialog.mobile.error.title"
        ▼
OTP sent → Verify OTP
        │
        ├── Success: "authentication.reset_password_otp.success.description"
        ├── User not found: "authentication.reset_password_otp.user.not.found"
        └── Dialog: "authentication.reset_password_otp.dialog.title"
```

### 2.3 Social Login Providers

| Provider | Plugin | Translation Key |
|----------|--------|----------------|
| Google | `google_sign_in_android` | "authentication.landing.continue.with.google" |
| Apple | `sign_in_with_apple` | "authentication.landing.continue.with.apple" |
| Facebook | `sign_in_with_facebook` (custom: `com.talabat.sign_in_with_facebook`) | "authentication.landing.continue.with.facebook" |
| Mobile OTP | Built-in | "authentication.landing.continue.with.mobile" |
| Email | Built-in | "authentication.landing.continue.with.email" |

**Social Login Flow:**

```
User taps social provider button
        │
        ▼
Provider SDK authentication
        │
        ├── Success: Provider ID token received
        │       │
        │       ▼
        │   API: POST /auth/social/login
        │       Body: { provider, id_token, device_info }
        │       │
        │       ├── Account exists → JWT tokens issued
        │       │
        │       └── Account doesn't exist → Linked account alert
        │           "authentication.linked.account.alert.title"
        │           "authentication.linked.account.alert.description"
        │           │
        │           ├── Create new account: "authentication.linked.account.create.account"
        │           └── Link to existing: "authentication.linked.account.send.verification"
        │               "authentication.linked.account.invalid.verification.ticket"
        │
        └── Failure: "authentication.failed.forbidden.user.error"
                     "authentication.failed.forbidden.user.error.description"
```

### 2.4 Skip Authentication (Guest Mode)

The app supports guest browsing via `"authentication.landing.skip"` / `"authentication.skip.button"`. Guest users can:

- Browse vendors and menus
- View search results and categories
- Read reviews and ratings

Guest users **cannot**:
- Place orders
- Save addresses
- Add payment methods
- Access order history
- Use wallet or BNPL features

The feature flag `exp_hide_skip_during_onboarding` can hide the skip button to force registration during onboarding.

---

## 3. Registration Flow

### 3.1 Full Registration

```
Sign Up Screen: "authentication.sign_up.title"
        │
        ├── Tab: "authentication.login_and_registration.tabs.sign_up"
        │   Title: "authentication.login_and_registration.title"
        │
        ├── Fields:
        │   ├── First Name: "authentication.sign_up.first.name"
        │   ├── Last Name: "authentication.sign_up.last.name"
        │   ├── Email: "authentication.sign_up.email.subtitle"
        │   │   Helper: "authentication.login_otp.register.email.helper.text"
        │   ├── Mobile: "authentication.sign_up.mobile.subtitle"
        │   │   Consent: "authentication.sign_up.mobile.consent.text"
        │   │   Consent message: "authentication.sign_up.mobile.consent.message"
        │   ├── Password: "authentication.sign_up.choose.password"
        │   │   Helper: "authentication.sign_up.password.helper"
        │   ├── Date of Birth: "authentication.sign_up.date.of.birth"
        │   └── Verification Code: "authentication.sign_up.verification.code"
        │
        ├── Legal:
        │   ├── Terms: "authentication.sign_up.terms.of.use"
        │   ├── Privacy: "authentication.sign_up.privacy.policy"
        │   └── Consent connector: "authentication.sign_up.by.creating.account"
        │                          "authentication.sign_up.and"
        │                          "authentication.sign_up.to.the"
        │
        ├── Offers opt-in: "authentication.sign_up.offers.check.box.title"
        │
        └── Submit: "authentication.sign_up.verify"
                    "authentication.sign_up.create.account"
```

### 3.2 Unified Email Login/Registration

The feature flag `ff_user_unified_email_login_and_registration_screen` merges the login and registration screens into a single unified flow, reducing confusion for users who aren't sure if they have an account.

### 3.3 FullPolygon Endpoint Migration

The feature flag `ff_user_fullpolygon_endpoint_migration` indicates a backend migration to a new unified authentication endpoint ("FullPolygon"), consolidating multiple legacy auth endpoints into a single API.

---

## 4. Session Management

### 4.1 JWT Token Structure

```
Access Token (short-lived):
{
  "sub": "user_uuid",
  "phone": "+971501234567",
  "email": "user@example.com",
  "country_code": "AE",
  "global_entity_id": "TB_AE",
  "roles": ["customer"],
  "pro_status": "active",
  "iat": 1700000000,
  "exp": 1700003600     // 1-hour expiry
}

Refresh Token (long-lived):
{
  "sub": "user_uuid",
  "device_id": "device_uuid",
  "iat": 1700000000,
  "exp": 1702588000     // 30-day expiry
}
```

### 4.2 Token Storage

| Token | Storage Location | Encryption |
|-------|-----------------|------------|
| Access token | `FlutterSharedPreferences` (key: `flutter.auth_token`) | AES-256 encrypted at rest |
| Refresh token | `token_secure_storage` plugin (native) | Hardware-backed keystore |
| Payment tokens | `payment_native_storage` plugin (native) | Hardware-backed keystore |
| Braze auth | Braze SDK internal | Braze-managed encryption |

### 4.3 Token Refresh Flow

```
Access token expires (401 response)
        │
        ▼
Client: POST /auth/token/refresh
Body: { refresh_token }
        │
        ├── Success → New access + refresh tokens issued
        │              All in-flight requests retried with new token
        │
        ├── Refresh token expired → Full re-authentication required
        │   "authentication.refresh.forbidden"
        │
        └── Rate limited → Backoff and retry
```

### 4.4 Multi-Device Sessions

Users can be logged in on multiple devices simultaneously. Each device maintains its own refresh token. Session invalidation events (password change, account lock) propagate to all devices via FCM push, forcing re-authentication.

---

## 5. Authorization Model

### 5.1 Roles

| Role | Scope | Permissions |
|------|-------|-------------|
| `guest` | Unauthenticated | Browse, search, view menus |
| `customer` | Authenticated user | Place orders, manage addresses, payments, view history |
| `pro_customer` | Pro subscriber | Free delivery, exclusive offers, family plan management |
| `vendor_admin` | Restaurant/store manager | Manage menu, accept/reject orders, update stock |
| `rider` | Delivery rider | Accept orders, update location, manage deliveries |
| `support_agent` | Customer support | View orders, issue refunds, chat with customers |
| `admin` | Platform administrator | Full system access, feature flag management |

### 5.2 Permission Matrix for Customer Role

| Resource | Read | Write | Conditions |
|----------|------|-------|------------|
| Own profile | Yes | Yes | — |
| Own addresses | Yes | Yes | Max 10 addresses |
| Own orders | Yes | No (cancel only) | Within cancellation window |
| Own payment methods | Yes | Yes (add/remove) | PCI-compliant tokenization |
| Own wallet | Yes | Yes (top-up/withdraw) | KYC-dependent limits |
| Own BNPL | Yes | Yes (payments) | Credit check passed |
| Vendor menus | Yes | No | — |
| Search | Yes | No | — |
| Reviews/ratings | Yes | Yes (own reviews) | One review per order |
| Rider chat | Yes | Yes | Active order only |

### 5.3 Authorization Middleware

API requests pass through authorization middleware that:

1. **Validates JWT** — Signature, expiry, issuer
2. **Checks role permissions** — Role-based access control per endpoint
3. **Validates country scope** — User can only access data in their `country_code`
4. **Rate limits** — Per-user and per-IP rate limiting
5. **Fraud check** — Shield Service + Incognia verification for sensitive operations

---

## 6. Security Measures

### 6.1 Device Fingerprinting

The `Shield Service` plugin (`com.talabat.shield.ShieldServicePlugin`) collects device-level signals for fraud detection:

- Device model and OS version
- Screen resolution and density
- Installed app fingerprint (hash)
- Network information
- Sensor calibration data
- Battery level and charging state

The feature flag `ff_killswitch_shield_tracking` provides a kill switch for this tracking if it causes issues.

### 6.2 Location-Based Verification

The `Incognia` SDK provides **location-based identity verification**:

- Creates a location behavioral fingerprint for each device
- Detects location spoofing and GPS manipulation
- Identifies suspicious patterns (e.g., account accessed from impossible locations)
- Config: Location collection enabled, installed apps collection disabled, logging disabled
- Kill switch: `ff_killswitch_incognia`

### 6.3 Bot Protection

`reCAPTCHA Enterprise` (`recaptcha_enterprise_flutter`) is integrated for:

- OTP request protection (prevent SMS flooding)
- Account creation protection (prevent mass registration)
- Login protection (prevent credential stuffing)
- Payment protection (prevent automated fraud)

### 6.4 VM/Emulator Detection

The feature flag `ff_is_vm_detection_enabled` enables detection of:

- Android emulators (Bluestacks, Nox, etc.)
- Virtual machines
- Rooted/jailbroken devices
- Frida instrumentation framework (also bundled in the APK for debugging)

When detected, the app may:
- Block authentication entirely
- Restrict to guest mode only
- Require additional verification steps

---

## 7. Account Management

### 7.1 Profile Management

| Field | Editable | Validation |
|-------|----------|------------|
| First name | Yes | Required, min 2 chars |
| Last name | Yes | Required, min 2 chars |
| Email | Yes | Valid email format, unique |
| Phone number | Yes (`ff_user_update_phone_number_enabled`) | E.164 format, unique, OTP verified |
| Date of birth | Yes | Must be 13+ years old |
| Gender | Yes | Male, Female, Optional |
| Password | Yes | Min 8 chars, complexity requirements |

Translation keys:
- `"account_info_title"`, `"account_info_edit"`, `"account_info_save"`
- `"account_info_first_name"`, `"account_info_last_name"`, `"account_info_email"`
- `"account_info_mobile_number"`, `"account_info_date_of_birth"`
- `"account_info_gender_optional"`, `"account_info_male"`, `"account_info_female"`
- `"account_info_talabat_reference_number"` (read-only, copyable)

### 7.2 Account Deletion

The translation key `"account_info_delete"` indicates the app supports account deletion (required by app store policies and GDPR-equivalent regulations in some markets). Deletion flow:

1. User requests account deletion
2. Confirmation dialog with consequences explanation
3. 30-day grace period (account deactivated but recoverable)
4. Permanent deletion after grace period
5. All PII purged from databases
6. Anonymized order data retained for analytics

### 7.3 Newsletter Opt-In

The `"account_info_subscription_news_letter"` and `"authentication.sign_up.offers.check.box.title"` keys indicate an optional marketing newsletter subscription, managed per consent requirements.

---

## 8. Cross-Country Authentication

### 8.1 Multi-Country Support

Users in different countries authenticate against country-specific auth endpoints:

| Country | Global Entity ID | Auth Domain |
|---------|-----------------|-------------|
| UAE | TB_AE | talabat.ae |
| Kuwait | TB_KW | talabat.com.kw |
| Egypt | HF_EG | hungerstation.com.eg / talabat.eg |
| Bahrain | TB_BH | talabat.bh |
| Oman | TB_OM | talabat.om |
| Qatar | TB_QA | talabat.qa |
| Saudi Arabia | TB_SA | talabat.sa |
| Jordan | TB_JO | talabat.jo |
| Iraq | TB_IQ | talabat.iq |

### 8.2 Country Switching

The `"address_list.change_country.title"` and `"address_list.change_country.description"` keys indicate users can switch countries, which:

1. Clears current cart (different vendor pools per country)
2. Updates API endpoint to new country's backend
3. Preserves authentication (SSO across country domains)
4. Migrates addresses and payment methods (if supported in new country)

---

## 9. Compliance & Privacy

### 9.1 Consent Management

The app implements explicit consent collection:

- **SMS consent**: `"authentication.sign_up.mobile.consent.text"` / `"authentication.sign_up.mobile.consent.message"`
- **Marketing consent**: `"authentication.sign_up.offers.check.box.title"`
- **Data processing**: `"authentication.sign_up.by.creating.account"` + terms/privacy links
- **AI chat disclaimer**: `"ai_chat_disclaimer"` — Explicit disclosure of OpenAI LLM usage and US data transfer

### 9.2 Data Processing Legal Basis

| Data Category | Legal Basis | Retention |
|---------------|-------------|-----------|
| Authentication credentials | Contract performance | Active + 30 days |
| Device fingerprints | Legitimate interest (fraud prevention) | 90 days |
| Location data | Consent | Session-based (Incognia) |
| Payment data | Contract performance | As required by PCI-DSS |
| Marketing preferences | Consent | Until withdrawn |
