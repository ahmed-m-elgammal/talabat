# 12 — Frontend Architecture

## 1. Overview

Talabat's mobile application is built on **Flutter**, Google's cross-platform UI framework, with a thin native Android layer for platform-specific integrations (payment SDKs, foreground services, push notifications). The architecture follows a **feature-first modular design** with clear separation between domain logic, data access, and presentation layers. The app supports both Google Play Services and Huawei Mobile Services (HMS) devices, with runtime detection and conditional feature activation.

The Flutter codebase is organized into **feature modules** (talabat, qcommerce_home, qcommerce_vendor_landing_page, qcommerce_categories_list, qcommerce_vendor_list, food_vendor_rating_domain) and **shared packages** (design_system, common_assets, l10n, fluid). This modular architecture enables independent feature development, testing, and deployment while maintaining a cohesive user experience.

---

## 2. Technology Stack

### 2.1 Core Framework

| Technology | Version | Purpose |
|-----------|---------|---------|
| Flutter | 3.x (Dart 2.x) | Cross-platform UI framework |
| Dart | 2.x | Programming language |
| Kotlin | 2.2.0 | Native Android layer |
| Android Min SDK | 21 | Minimum Android version (5.0 Lollipop) |
| Android Target SDK | Latest | Target Android version |

### 2.2 State Management

Based on the app's complexity and feature structure, the state management follows a **BLoC/Cubit pattern** (common in Delivery Hero's Flutter codebase):

```
UI Layer (Widgets)
        │
        ├── BLoC/Cubit (Business Logic Component)
        │   ├── Handles user interactions
        │   ├── Manages state transitions
        │   ├── Delegates to use cases
        │   └── Emits new states
        │
        ├── Use Cases (Domain Layer)
        │   ├── Contain business rules
        │   ├── Coordinate between repositories
        │   └── Transform data for presentation
        │
        └── Repositories (Data Layer)
            ├── Abstract interfaces
            ├── Concrete implementations
            ├── Remote data sources (API clients)
            └── Local data sources (SQLite, SharedPreferences)
```

### 2.3 Dependency Injection

The app uses **Dagger/Hilt** on the native side (evidenced by `DaggerFramesDIComponent` in Checkout.com SDK) and likely **GetIt/Injectable** or a similar DI framework on the Flutter side for managing dependencies across feature modules.

---

## 3. Project Structure

### 3.1 Flutter Module Architecture

```
talabat-app/
├── lib/
│   ├── main.dart                          # App entry point
│   ├── app.dart                           # MaterialApp configuration
│   ├── core/                              # Shared core utilities
│   │   ├── network/                       # API client, interceptors
│   │   ├── storage/                       # Local storage abstractions
│   │   ├── analytics/                     # Perseus integration
│   │   ├── navigation/                    # Router configuration
│   │   ├── theme/                         # App theme definitions
│   │   └── utils/                         # Helpers, extensions
│   ├── features/                          # Feature modules
│   │   ├── auth/                          # Authentication
│   │   ├── home/                          # Home screen
│   │   ├── search/                        # Search & discovery
│   │   ├── vendor/                        # Vendor listing & detail
│   │   ├── menu/                          # Menu browsing
│   │   ├── cart/                          # Cart management
│   │   ├── checkout/                      # Checkout flow
│   │   ├── order/                         # Order tracking & history
│   │   ├── payment/                       # Payment methods
│   │   ├── wallet/                        # talabat Pay wallet
│   │   ├── bnpl/                          # Buy Now Pay Later
│   │   ├── subscription/                  # talabat Pro
│   │   ├── rewards/                       # Loyalty points
│   │   ├── dineout/                       # DineOut reservations
│   │   ├── pharmacy/                      # Pharmacy vertical
│   │   ├── qcommerce/                     # Quick commerce (grocery)
│   │   ├── chat/                          # Customer support & AI
│   │   ├── notifications/                 # Notification center
│   │   ├── profile/                       # User profile
│   │   └── settings/                      # App settings
│   └── di/                                # Dependency injection setup
│
├── packages/                              # Shared packages
│   ├── design_system/                     # MM3 design system
│   ├── common_assets/                     # Shared images, icons
│   ├── l10n/                              # Localization
│   └── fluid/                             # Animation utilities
│
├── assets/
│   ├── flutter_assets/                    # Flutter asset bundle
│   ├── i18n/                              # Translation files
│   ├── fonts/                             # Custom fonts
│   ├── animations/                        # Lottie/Rive animations
│   └── images/                            # Static images
│
└── test/                                  # Test suites
    ├── unit/                              # Unit tests
    ├── widget/                            # Widget tests
    ├── integration/                       # Integration tests
    └── golden/                            # Golden image tests (golden_toolkit)
```

### 3.2 Feature Module Internal Structure

Each feature module follows a consistent internal structure:

```
feature/
├── presentation/
│   ├── pages/                             # Full-screen pages
│   ├── widgets/                           # Feature-specific widgets
│   ├── bloc/                              # BLoCs and Cubits
│   │   ├── feature_bloc.dart
│   │   ├── feature_event.dart
│   │   └── feature_state.dart
│   └── mappers/                           # Domain → Presentation mappers
├── domain/
│   ├── entities/                          # Business objects
│   ├── repositories/                      # Abstract repository interfaces
│   └── usecases/                          # Use case classes
├── data/
│   ├── models/                            # API/DTO models
│   ├── repositories/                      # Repository implementations
│   ├── datasources/                       # Remote and local data sources
│   └── mappers/                           # Model → Entity mappers
└── di/                                    # Module-specific DI
```

---

## 4. Design System (MM3)

### 4.1 Design System Package

The `design_system` package implements Talabat's **MM3 design system**, providing a consistent UI component library across all feature modules.

**Typography:**

| Font Family | Weights | Languages |
|-------------|---------|-----------|
| TTCommonsPro | 400, 600, 700, 800, 900 | English |
| NotoSansArabic | 400, 500, 700, 800, 900 | Arabic |
| Ubuntu Sans | 400 | English (secondary) |
| MaterialIcons | Regular | Icons |

**Icon System (200+ Icons):**

The design system includes 200+ custom icons covering all app features:

| Icon Category | Examples |
|--------------|----------|
| Navigation | `ds_home`, `ds_search`, `ds_orders`, `ds_profile` |
| Verticals | `ds_food`, `ds_grocery`, `ds_pharmacy`, `ds_flowers`, `ds_dine_out` |
| Payments | `ds_wallet`, `ds_credit_card`, `ds_apple_pay`, `ds_google_pay` |
| Regional Payments | `ds_meeza`, `ds_knet`, `ds_boubyan`, `ds_sadad`, `ds_benefit_pay`, `ds_snap` |
| Delivery | `ds_delivery`, `ds_pickup`, `ds_contactless_delivery`, `ds_delivery_tgo` |
| Pro | `ds_pro_tag_neutral`, `ds_pro_tag_filled`, `ds_pro_tag_white_filled` |
| Features | `ds_ai`, `ds_tgo_badge`, `ds_talabat_pickers`, `ds_carbon_neutral_filled` |
| Actions | `ds_add`, `ds_remove`, `ds_favorite`, `ds_share`, `ds_filter` |
| DineOut | `ds_live_screening`, `ds_live_music`, `ds_serves_alcohol`, `ds_serves_sheesha` |
| Misc | `ds_accessible`, `ds_charity`, `ds_recycle`, `ds_flash_sale_filled`, `ds_deals_for_you` |

### 4.2 Theming

The app supports **light and dark themes** with RTL (right-to-left) layout for Arabic:

```dart
// Pseudocode for theme configuration
class TalabatTheme {
  // Primary brand color: Talabat orange (#FF5A00)
  static const primaryColor = Color(0xFFFF5A00);
  
  // Pro brand color
  static const proColor = Color(0xFF...);
  
  // Dark theme support
  static ThemeData lightTheme = ThemeData(...);
  static ThemeData darkTheme = ThemeData(...);
  
  // RTL support
  static TextDirection textDirection = TextDirection.rtl; // for Arabic
}
```

---

## 5. Native Layer (Android)

### 5.1 Native Code Purpose

The native Android layer is intentionally thin, handling only platform-specific functionality that cannot be implemented in Flutter:

| Native Component | Package | Purpose |
|-----------------|---------|---------|
| `TalabatApplication` | `com.talabat` | App initialization, Braze config, notification channels |
| `TalabatFirebaseMessagingService` | `com.talabat` | Push notification handling (extends Braze) |
| `FlutterLandingScreen` | `com.talabat` | Deep link landing activity |
| `LiveNotificationService` | `com.talabat` | Foreground service for order tracking |
| `SecurePaymentRedirectionWebViewActivity` | `com.talabat` | WebView for 3DS/redirect payments |
| `HyperPayPlugin` | `com.talabat.hyper_pay` | OPPWA payment SDK bridge |
| `CardTokenizationPlugin` | `com.talabat.card_tokenization` | Checkout.com tokenization bridge |
| `GooglePayPlugin` | `com.talabat.googlepay` | Google Pay bridge |
| `BenefitPayPlugin` | `com.talabat.benefit_pay` | Bahrain BenefitPay bridge |
| `ZaincashPlugin` | `com.talabat.zaincash` | Iraq ZainCash bridge |
| `PaymentNativeStoragePlugin` | `com.talabat.payment_native_storage` | Secure payment data storage |
| `TokenSecureStoragePlugin` | `com.talabat.token_secure_storage` | Secure token storage |
| `ShieldServicePlugin` | `com.talabat.shield` | Device fingerprinting |
| `PerformanceAnalyticsPlugin` | `com.talabat.performance_analytics` | Performance tracking |
| `PerformanceFlutterKitPlugin` | `com.talabat.performance.kit` | Performance kit |
| `PerseusAnalyticsPlugin` | `com.talabat.perseus` | Analytics event tracking |
| `MobileServicesTypePlugin` | `com.talabat.mobile_services_type` | Google/HMS detection |
| `AppIconSwitcherPlugin` | `com.talabat.app_icon_switcher` | Dynamic app icon |
| `SignInWithFacebookPlugin` | `com.talabat.sign_in_with_facebook` | Facebook auth bridge |

### 5.2 Plugin Registration

All native plugins are registered through Flutter's `GeneratedPluginRegistrant`:

```kotlin
// Simplified from decompiled GeneratedPluginRegistrant
class GeneratedPluginRegistrant {
    fun registerWith(registry: FlutterPluginRegistry) {
        // Auth plugins
        GoogleSignInPlugin.registerWith(...)
        SignInWithApplePlugin.registerWith(...)
        SignInWithFacebookPlugin.registerWith(...)
        SmsAutofillPlugin.registerWith(...)
        RecaptchaEnterprisePlugin.registerWith(...)
        
        // Payment plugins
        HyperPayPlugin.registerWith(...)
        CardTokenizationPlugin.registerWith(...)
        GooglePayPlugin.registerWith(...)
        BenefitPayPlugin.registerWith(...)
        ZaincashPlugin.registerWith(...)
        PaymentNativeStoragePlugin.registerWith(...)
        TokenSecureStoragePlugin.registerWith(...)
        
        // Analytics & monitoring
        PerseusAnalyticsPlugin.registerWith(...)
        PerformanceAnalyticsPlugin.registerWith(...)
        PerformanceFlutterKitPlugin.registerWith(...)
        NewRelicMobilePlugin.registerWith(...)
        SentryFlutterPlugin.registerWith(...)
        
        // Maps & location
        GoogleMapsPlugin.registerWith(...)
        HuaweiMapPlugin.registerWith(...)
        GeolocatorPlugin.registerWith(...)
        
        // Push notifications
        BrazePlugin.registerWith(...)
        
        // Fraud prevention
        ShieldServicePlugin.registerWith(...)
        IncogniaPlugin.registerWith(...)
        
        // Other platform plugins
        FirebaseCorePlugin.registerWith(...)
        FirebaseDatabasePlugin.registerWith(...)
        FirebaseAnalyticsPlugin.registerWith(...)
        FirebaseCrashlyticsPlugin.registerWith(...)
        FirebasePerformancePlugin.registerWith(...)
        SqflitePlugin.registerWith(...)
        SharedPreferencesPlugin.registerWith(...)
        // ... 60+ plugins total
    }
}
```

### 5.3 Dual Platform Services (Google + HMS)

The `mobile_services_type` plugin detects available platform services at runtime:

```kotlin
// Pseudocode from decompiled sources
fun hasGooglePlayServices(): Boolean {
    return GoogleApiAvailability.getInstance()
        .isGooglePlayServicesAvailable(context) == ConnectionResult.SUCCESS
}

fun hasHuaweiMobileServices(): Boolean {
    return HuaweiApiAvailability.getInstance()
        .isHuaweiMobileServicesAvailable(context) == ConnectionResult.SUCCESS
}
```

**Conditional Feature Activation:**

| Feature | Google Play Services | Huawei Mobile Services |
|---------|--------------------|-----------------------|
| Maps | Google Maps (`google_maps_flutter_android`) | Huawei Maps (`huawei_map`) |
| Push | FCM | HMS Push Kit |
| Auth | Google Sign-In | Huawei ID (via HMS) |
| Braze API Key | `f880a0a8-df23-4a78-80ee-096cfd56ea67` | `0d387798-0f0b-43b6-8610-fea0ce9fe7fc` |

---

## 6. Navigation & Routing

### 6.1 Navigation Architecture

The app uses **declarative routing** (likely `go_router` or a custom router) with deep link support:

```
/                           → Home
/search                     → Search
/food                       → Food vertical
/grocery                    → Grocery vertical
/pharmacy                   → Pharmacy vertical
/vendor/{id}                → Vendor detail (menu)
/vendor/{id}/item/{id}      → Item detail
/cart                       → Cart
/checkout                   → Checkout
/orders                     → Order history
/orders/{id}                → Order detail
/orders/{id}/tracking       → Live tracking
/profile                    → User profile
/wallet                     → Wallet dashboard
/bnpl                       → BNPL dashboard
/subscriptions              → Pro management
/rewards                    → Rewards
/dineout                    → DineOut listings
/dineout/booking/{id}       → Booking detail
/settings                   → Settings
/auth/login                 → Login
/auth/signup                → Registration
```

### 6.2 Deep Linking

The `app_links` plugin (`com.llfbandit.app_links`) handles incoming deep links:

- **Scheme**: `talabat://`
- **Universal Links**: `https://talabat.ae/`, `https://talabat.com.kw/`, etc.
- **Push deep links**: Braze-configured deep links with `FlutterLandingScreen` as back stack activity
- **Feature flag**: Braze `setPushDeepLinkBackStackActivityEnabled(true)` ensures proper navigation stack

---

## 7. Localization

### 7.1 Supported Languages

| Language | File | Direction |
|----------|------|-----------|
| English | `en.json` | LTR |
| Arabic | `ar.json` | RTL |
| Arabic (UAE) | `ar-AE.json` | RTL |
| Kurdish Sorani | `ckb.json` | RTL |

### 7.2 RTL Support

The design system and all feature modules support right-to-left layout:

- Bidirectional text rendering (Arabic + English mixed content)
- Mirrored navigation patterns (back arrow direction, swipe gestures)
- Font families: NotoSansArabic for Arabic, TTCommonsPro for English
- Currency formatting: AED, BHD, EGP, IQD, JOD, KWD, OMR, QAR, SAR

### 7.3 Dialect Support

The feature flag `ff_qcommerce_dialects` enables dialect-specific translations for grocery products, recognizing that Arabic dialects vary significantly across the 9 countries (e.g., "bread" is "خبز" in Gulf Arabic but "عيش" in Egyptian Arabic).

---

## 8. Performance Optimization

### 8.1 App Startup

| Phase | Duration (High-End) | Duration (Low-End) |
|-------|---------------------|-------------------|
| Native startup | 200ms | 500ms |
| Flutter engine init | 300ms | 600ms |
| `appStartToInteractiveFlutter` | 1191ms | 2000ms |
| `TTI_HOME` | 726ms | 2000ms |

**Optimizations:**
- Baseline profiles (`baseline.prof` + `baseline.profm`) for ART pre-compilation
- `AppStartupInitializer` for lazy SDK initialization
- Deferred component loading for non-critical features
- Image caching with Coil (native) and cached_network_image (Flutter)

### 8.2 Performance Monitoring

| Tool | Plugin | Metric |
|------|--------|--------|
| Delivery Hero Performance Kit | `com.talabat.performance.kit` | Screen TTI, frame metrics |
| Firebase Performance | `firebase_performance` | Network traces, screen traces |
| New Relic | `newrelic_mobile` | APM, crash analysis |
| Custom TTI tracking | `performance_analytics` | Per-screen TTI with device-class thresholds |
| Hang observer | `ff_hang_observer_enable` | Detect UI thread hangs |

### 8.3 Image Optimization

- **Coil** (native Android): Image loading with memory/disk caching
- **Cached network images** (Flutter): Transparent HTTP caching
- **CDN optimization**: Dynamic image resizing via URL parameters
- **WebP format**: Preferred format for all images
- **Lazy loading**: Images loaded on scroll, not pre-loaded

### 8.4 HTTP/2 Experimentation

The feature flag `exp_platform_http2` enables HTTP/2 for API communication:

- Multiplexed requests over single connection
- Server push for proactive data delivery
- Header compression (HPACK)
- Connection coalescing across domains

---

## 9. Testing Architecture

### 9.1 Test Types

| Type | Framework | Scope |
|------|-----------|-------|
| Unit tests | Flutter test | BLoCs, use cases, repositories, mappers |
| Widget tests | Flutter test | Individual widgets, screens |
| Integration tests | `integration_test` plugin | Full user flows across features |
| Golden tests | `golden_toolkit` package | Pixel-perfect UI regression tests |
| Native tests | JUnit/MockK | Native plugin unit tests |

### 9.2 Mock Data Packages

The Flutter packages include mock data modules for development and testing:

| Package | Content |
|---------|---------|
| `qcommerce_home` | Home screen mock data |
| `qcommerce_vendor_list` | Vendor list mock data |
| `qcommerce_categories_list` | Category list mock data |
| `qcommerce_vendor_landing_page` | VLP mock data |
| `food_vendor_rating_domain` | Rating domain test data |

---

## 10. Build & Release

### 10.1 Build Configuration

| Parameter | Value |
|-----------|-------|
| Package name | `com.talabat` |
| Min SDK | 21 |
| Target SDK | Latest |
| Build tools | Gradle + Flutter build |
| Code shrinking | R8/ProGuard (obfuscated output confirms) |
| Signing | Release keystore (SHA-256 in `stamp-cert-sha256`) |
| Baseline profiles | `baseline.prof` (26.6KB) + `baseline.profm` (853B) |

### 10.2 App Size

| Component | Size |
|-----------|------|
| Total APK | ~45 MB |
| classes.dex (x3) | ~23 MB |
| resources.arsc | ~792 KB |
| Flutter assets | ~500 KB |
| Native libraries | Minimal (Flutter snapshot-based) |
| Assets (images, fonts) | ~5 MB |
| res/ (Android resources) | ~15 MB |
