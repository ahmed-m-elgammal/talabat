# 14 — Payment System

## 1. Overview

Talabat's payment system is a multi-gateway, multi-method financial infrastructure supporting 9 MENA countries with diverse payment preferences ranging from international credit cards to regional mobile wallets. The system processes payments through two primary gateways — **Checkout.com** (for card tokenization) and **HyperPay/OPPWA** (for full checkout flows) — along with native integrations for Apple Pay, Google Pay, and country-specific payment methods (BenefitPay in Bahrain, ZainCash in Iraq, STC Pay in Saudi Arabia, Meeza in Egypt, KNET in Kuwait, Sadad in Saudi Arabia).

Beyond traditional payments, the system encompasses Talabat's fintech suite: **talabat Pay** (digital wallet), **PostPaid** (Buy Now Pay Later / BNPL), and **co-branded credit cards** (ADCB in UAE, CIB in Egypt). This makes the payment system one of the most complex subsystems in the platform, requiring PCI-DSS Level 1 compliance, 3D Secure 2 authentication, and sophisticated fraud prevention.

---

## 2. Payment Methods

### 2.1 Supported Payment Methods by Country

| Payment Method | UAE | KW | BH | OM | QA | SA | EG | JO | IQ |
|---------------|-----|----|----|----|----|----|----|----|----| 
| Credit/Debit Card (Visa, MC, Amex) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Apple Pay | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | — |
| Google Pay | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | — | — |
| talabat Pay (Wallet) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| PostPaid (BNPL) | ✅ | ✅ | — | — | — | ✅ | ✅ | — | — |
| STC Pay | — | — | — | — | — | ✅ | — | — | — |
| BenefitPay | — | — | ✅ | — | — | — | — | — | — |
| ZainCash | — | — | — | — | — | — | — | — | ✅ |
| Meeza | — | — | — | — | — | — | ✅ | — | — |
| KNET | — | ✅ | — | — | — | — | — | — | — |
| Sadad | — | — | — | — | — | ✅ | — | — | — |
| Cash on Delivery | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Co-branded CC (ADCB) | ✅ | — | — | — | — | — | — | — | — |
| Co-branded CC (CIB) | — | — | — | — | — | — | ✅ | — | — |

### 2.2 Card Tokenization

Cards are tokenized to avoid storing raw card data on Talabat's servers (PCI-DSS compliance):

| Gateway | Tokenization Method | Storage |
|---------|-------------------|---------|
| Checkout.com | Client-side tokenization → `api.checkout.com/tokens` | Token stored in `payment_native_storage` (native secure storage) |
| HyperPay/OPPWA | Server-side registration → `eu-prod.oppwa.com/registration` | Checkout ID stored in `token_secure_storage` |

**Checkout.com Tokenization Flow:**
```
1. Customer enters card details in Frames SDK UI
2. SDK validates: Luhn check, expiry, CVV via validators
3. SDK sends to Checkout.com: POST https://api.checkout.com/tokens
4. Checkout.com returns: token (tok_xxx) + card fingerprint
5. Talabat backend stores token reference (not card data)
6. Future payments use token for recurring charges
```

### 2.3 3D Secure 2 Authentication

Both gateways support 3DS2 for strong customer authentication:

**Checkout.com 3DS Flow:**
```
Payment request with card token
        │
        ▼
Checkout.com API: assess 3DS requirement
        │
        ├── Frictionless (no challenge)
        │   └── Payment approved immediately
        │
        └── Challenge required
            ├── ThreeDSExecutor initiates challenge
            ├── ThreeDSWebView displays challenge UI
            ├── Customer completes authentication (OTP, biometric)
            └── Payment completed or failed
```

**HyperPay 3DS2 Flow:**
```
Payment request with checkout ID
        │
        ▼
OPPWA: ThreeDS2TransactionManager
        │
        ├── APP flow (native SDK challenge)
        │   └── nsoftware.ipworks3ds SDK handles challenge
        │
        ├── APPONLY flow
        │   └── Challenge without fallback
        │
        └── WEB flow (fallback)
            └── SecurePaymentRedirectionWebViewActivity
                └── Redirect to issuer's challenge page
```

---

## 3. Payment Gateway Architecture

### 3.1 Dual-Gateway Strategy

Talabat uses two payment gateways with intelligent routing:

| Gateway | Primary Use | SDK | Package |
|---------|------------|-----|---------|
| **Checkout.com** | Card tokenization, risk assessment | Frames SDK 4.2.6 | `com.checkout` |
| **HyperPay (OPPWA)** | Full checkout flow, regional methods | Mobile Connect SDK | `com.oppwa.mobile.connect` |

**Gateway Routing Logic:**
```
Payment request
        │
        ├── Card payment?
        │   ├── New card → Checkout.com tokenization
        │   ├── Saved token → Original gateway (token affinity)
        │   └── BIN-specific routing → Gateway matching card range
        │
        ├── Apple Pay? → Native Apple Pay framework
        ├── Google Pay? → Google Pay plugin
        ├── Wallet? → Internal wallet service
        ├── BNPL? → Internal BNPL service
        └── Regional method? → HyperPay (supports all regional brands)
```

### 3.2 Checkout.com SDK Details

**Configuration:**
```kotlin
CheckoutApiServiceFactory.create(
    publicKey = "pk_xxx",
    environment = Environment.PRODUCTION, // or SANDBOX
    context = applicationContext
)
```

**API Endpoints:**

| Environment | URL |
|-------------|-----|
| Tokenization (Production) | `https://api.checkout.com/tokens` |
| Tokenization (Sandbox) | `https://api.sandbox.checkout.com/tokens` |
| Risk Device Data (Production) | `https://risk.checkout.com` |
| Risk Fingerprint (Production) | `https://fpjs.checkout.com` |
| Event Logger (Production) | `https://cloudevents.integration.checkout.com/logging` |

**Supported Token Types:**
- `card` — Credit/debit card tokenization
- `cvv` — CVV-only tokenization for recurring payments
- `googlepay` — Google Pay tokenization

**Risk SDK Integration:**
The Checkout.com Risk SDK provides device fingerprinting and fraud scoring:
- `RiskSDKInternalConfigImpl` — Device data collection
- `DeviceDataApi` — Send device data for risk assessment
- `RiskSdkUseCase` — Orchestrate risk check before payment

### 3.3 HyperPay/OPPWA SDK Details

**Configuration:**
```kotlin
val provider = Connect.getProvider(context, ProviderMode.LIVE)
// ProviderMode.TEST for testing
```

**API Endpoints:**

| Environment | URL |
|-------------|-----|
| LIVE | `https://eu-prod.oppwa.com:443` |
| TEST | `https://eu-test.oppwa.com:443` |

**API Routes:**
- `/payment` — Submit transaction
- `/registration` — Register card token
- `/omnitoken` — Omni token creation

**Supported Payment Brands (40+):**
AMEX, VISA, MASTERCARD, GOOGLEPAY, SAMSUNGPAY, APPLEPAY, PAYPAL, KLARNA (installments/invoice/paylater/paynow/sliceit), STC_PAY, IDEAL, GIROPAY, BLIK, MBWAY, AMAZONPAY, AFFIRM, AFTERPAY_PACIFIC, CLEARPAY, BANCONTACT_LINK, CHINAUNIONPAY, DANKORT, DIRECTDEBIT_SEPA, CASH_APP_PAY, IKANOOI, RATEPAY_INVOICE, SOFORTUEBERWEISUNG, INICIS, ONEY

**Payment Parameters:**
```kotlin
CardPaymentParams(
    checkoutId = "uuid",
    paymentBrand = "VISA",
    cardNumber = "4242...",
    cardHolder = "Ahmed Al-Rashid",
    cardExpiryMonth = "12",
    cardExpiryYear = "2028",
    cardCVV = "123",
    shopperResultUrl = "talabat://payment-result"
)
```

---

## 4. Payment Flow

### 4.1 Standard Card Payment Flow

```
Customer selects "Pay with Card"
        │
        ├── Saved card?
        │   ├── YES → Select saved card → CVV entry (if required)
        │   └── NO → Card entry form (Checkout.com Frames SDK)
        │       ├── Card number validation (Luhn + brand detection)
        │       ├── Expiry date validation
        │       ├── CVV validation
        │       └── Cardholder name entry
        │
        ▼
Tokenize card (Checkout.com or HyperPay)
        │
        ▼
Risk assessment
        ├── Checkout.com Risk SDK (device fingerprint)
        ├── Shield Service (device reputation)
        └── Incognia (location verification)
        │
        ▼
Payment authorization request
        │
        ├── 3DS required?
        │   ├── YES → Challenge flow → Customer authenticates
        │   └── NO → Direct authorization
        │
        ▼
Authorization result
        │
        ├── Success → Payment authorized, order confirmed
        │   └── Capture payment (immediate for food, delayed for grocery)
        │
        └── Failure → Error handling
            ├── Insufficient funds → "Contact your card issuer"
            ├── Card declined → "Try a different payment method"
            ├── 3DS failed → "Authentication failed, please try again"
            └── Gateway error → "Payment processing error, retry"
```

### 4.2 Apple Pay Flow

```
Customer selects "Pay with Apple Pay"
        │
        ▼
Apple Pay sheet presented
        │
        ├── Card selection (from device Wallet)
        ├── Authentication (Face ID / Touch ID / Passcode)
        │
        ▼
Payment token generated
        │
        ▼
Backend processes Apple Pay token
        │
        └── Order confirmed
```

**Apple Pay Contexts:**
- `"apple_pay_order_title"` — Standard order payment
- `"apple_pay_subscribe_title"` — Pro subscription payment
- `"apple_pay_in_store_title"` — In-store DineOut payment
- `"apple_pay_continue_title"` — Continue with Apple Pay

### 4.3 Wallet (talabat Pay) Flow

```
Customer selects "Pay with talabat Pay"
        │
        ├── Check wallet balance
        │   ├── Sufficient balance → Deduct and confirm
        │   └── Insufficient balance → Top-up prompt
        │       ├── Enter top-up amount
        │       ├── Select payment method for top-up
        │       ├── Top-up processed
        │       └── Payment proceeds with new balance
        │
        ▼
Order confirmed with wallet payment
```

### 4.4 BNPL (PostPaid) Flow

```
Customer selects "Pay with PostPaid"
        │
        ├── Check BNPL limit
        │   ├── Within limit → Proceed
        │   └── Over limit → "Insufficient balance" info
        │
        ├── Display terms
        │   "By placing order you agree" (bnpl.pay.later.by.placing.order.you.agree)
        │   Terms and conditions link
        │
        ▼
Order confirmed, payment due in 30 days
        │
        ├── Installment created
        │   ├── Payment due date displayed
        │   ├── Auto-payment method (if configured)
        │   └── Payment day preference (if configured)
        │
        └── Dashboard updated
            ├── Available balance decreased
            └── Upcoming payment added
```

---

## 5. BNPL (PostPaid) System

### 5.1 PostPaid Dashboard

The BNPL dashboard provides comprehensive payment management:

| Section | Content |
|---------|---------|
| Available balance | Current spending limit remaining |
| Total limit | Maximum credit limit |
| Total due | Sum of all unpaid installments |
| Upcoming payments | Scheduled payment list with due dates |
| Payment history | Completed payment records |
| Overdue alert | Red banner for overdue payments |

**Translation keys:**
- `bnpl.dashboard.available.balance` / `bnpl.dashboard.available.balance.remix`
- `bnpl.dashboard.total.limit` / `bnpl.dashboard.total.due`
- `bnpl.dashboard.upcoming.section.header.title` / `bnpl.dashboard.upcoming.section.header.title.remix`
- `bnpl.dashboard.overdue.alert.header.title` / `bnpl.dashboard.overdue.description`
- `bnpl.dashboard.payment.required`

### 5.2 Rewind Feature

The **Rewind** feature allows customers to retroactively convert a previously paid order to PostPaid:

```
1. Customer views order that was paid with card
2. Rewind option displayed: "Switch to PostPaid"
3. Customer confirms:
   - Original payment method is refunded
   - Order amount is added to BNPL balance
   - Payment due date set to 30 days from now
4. Processing bottom sheet displayed
5. Success confirmation
```

### 5.3 Multi-Order Payment

Customers can pay multiple BNPL installments at once:

```
1. Select multiple overdue/upcoming payments
2. Total amount calculated
3. Select payment method
4. Process payment for all selected orders
5. Update all installment statuses
```

### 5.4 BNPL Installment States

| State | Translation Key | Visual |
|-------|----------------|--------|
| Paid | `bnpl_installment_paid` | Green checkmark |
| Unpaid | `bnpl_installment_unpaid` | Gray dot |
| Overdue | `bnpl_installment_overdue` | Red warning |
| Cancelled | `bnpl_installment_cancelled` | Gray strikethrough |
| Failed | `bnpl_installment_failed` | Red X |
| Processing | `bnpl_transaction_Processing` | Spinning indicator |
| Refunded | `bnpl_installment_refunded` | Blue arrow |
| Paused | `bnpl_order_paused` | Yellow pause |

---

## 6. Co-Branded Credit Cards

### 6.1 ADCB talabat Credit Card (UAE)

| Feature | Value |
|---------|-------|
| Annual fee | Free for life |
| Welcome bonus | AED 500 |
| Monthly talabat credit | Up to AED 350 |
| Minimum salary | AED 5,000 |
| Minimum age | 21 years |
| ID required | Emirates ID |
| Application | In-app (fully digital) |

**Application Flow:**
1. Personal information
2. Address details
3. Contact information
4. OTP verification
5. Bank review (under review status)
6. Approval/rejection notification

### 6.2 CIB talabat Credit Card (Egypt)

| Feature | Value |
|---------|-------|
| Welcome bonus | Up to EGP 2,000 |
| Monthly talabat credit | Up to EGP 1,200 |
| ID required | National ID with factory number |
| OTP verification | Required |

---

## 7. Security & Compliance

### 7.1 PCI-DSS Compliance

| Requirement | Implementation |
|-------------|---------------|
| No card data storage | Card numbers never touch Talabat servers; tokenization at gateway |
| Encryption in transit | TLS 1.2+ for all payment API calls |
| Encryption at rest | Payment tokens encrypted in hardware-backed keystore |
| Access control | Payment service isolated in separate network segment |
| Logging | Card data masked in all logs (showing only last 4 digits) |
| Penetration testing | Annual third-party pen tests |

### 7.2 Fraud Prevention

| Layer | Tool | Function |
|-------|------|----------|
| Device fingerprinting | Checkout.com Risk SDK | Device identification and reputation scoring |
| Device fingerprinting | Shield Service (`shield_service_plugin`) | Custom device fingerprinting and VM detection |
| Location verification | Incognia (`incognia_flutter`) | Location-based identity verification |
| Bot protection | reCAPTCHA Enterprise | Automated attack prevention |
| Card security info | `card_security_information_*` translation keys | Customer education about card security |

### 7.3 Secure Storage

| Plugin | Purpose | Encryption |
|--------|---------|------------|
| `payment_native_storage` | Store payment method references | Android Keystore (hardware-backed) |
| `token_secure_storage` | Store auth/payment tokens | Android Keystore (hardware-backed) |
| `FlutterSharedPreferences` | General app preferences (non-sensitive) | Encrypted at rest |

---

## 8. Payment Error Handling

### 8.1 Error Categories

| Category | User Message | Action |
|----------|-------------|--------|
| Card declined | "Your card was declined" | Suggest different payment method |
| Insufficient funds | "Insufficient funds" | Suggest wallet top-up or different card |
| 3DS failed | "Authentication failed" | Retry with same or different card |
| Gateway timeout | "Payment is taking longer than expected" | Check order status, don't retry |
| Duplicate payment | "Payment already processed" | Redirect to order tracking |
| Invalid BIN voucher | "This voucher requires a specific card type" | Change payment method or remove voucher |
| CVV error | "Incorrect CVV" | Re-enter CVV |
| Expired card | "Your card has expired" | Add new card |

### 8.2 BIN-Based Voucher Validation

The feature flag `ff_pay_button_charge_warning` and BIN voucher error keys (`order.experience.checkout.binVoucher.error.*`) indicate that certain vouchers are restricted to specific card BIN ranges:

```
Voucher "VISA20" = 20% off with VISA cards only
        │
        ├── Customer applies voucher
        ├── Customer selects Mastercard
        ├── BIN check: Voucher requires VISA (4xxx)
        ├── Error: "This voucher requires a VISA card"
        ├── Primary action: "Change payment method"
        └── Secondary action: "Remove voucher"
```

### 8.3 Payment Retry Logic

```
Payment failed
        │
        ├── Retryable error? (network timeout, 3DS cancel)
        │   ├── YES → Allow retry with same method
        │   └── NO → Suggest alternative method
        │
        ├── Max retries (3) exceeded?
        │   └── YES → Show all available payment methods
        │
        └── Subscription payment retry (Pro)
            ├── "subscription.payment.retry" button
            └── Retry with same or different card
```

---

## 9. Payment Analytics

### 9.1 Tracked Events

| Event | Properties | Purpose |
|-------|-----------|---------|
| `payment_method_selected` | method_type, is_default, is_new_card | Method preference analysis |
| `payment_initiated` | order_id, amount, currency, gateway, method | Payment funnel tracking |
| `payment_3ds_challenged` | order_id, gateway, challenge_type | 3DS friction analysis |
| `payment_completed` | order_id, amount, gateway, latency_ms | Success rate monitoring |
| `payment_failed` | order_id, error_code, gateway, method | Failure analysis |
| `payment_refunded` | order_id, amount, reason, method | Refund rate tracking |
| `card_tokenized` | gateway, card_brand, is_new | Tokenization success rate |
| `wallet_topup` | amount, payment_method | Wallet adoption tracking |
| `bnpl_order_placed` | order_id, amount, available_limit | BNPL utilization |
| `bnpl_payment_made` | installment_id, amount, is_auto | BNPL repayment tracking |

### 9.2 Payment Performance Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Payment success rate | > 98% | Successful payments / initiated payments |
| 3DS completion rate | > 90% | 3DS challenges completed / initiated |
| Payment latency (p99) | < 5s | Time from payment initiation to gateway response |
| Gateway uptime | 99.99% | Available / total time |
| Tokenization success rate | > 99% | Tokens created / tokenization attempts |
