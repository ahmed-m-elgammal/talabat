# Talabat APK - Critical Patch Guide
**Date:** 2026-05-18T18:46:14.904097+00:00
**Target:** Talabat APK v13.58.1

> **URGENCY:** All 4 exploits are confirmed. Patches should be applied immediately.
> The hardcoded refresh token (EXP-01) should be revoked BEFORE the next release.

---
## EXP-01: Hardcoded OAuth2 Refresh Token (CRITICAL - CVSS 9.8)

### Immediate Actions (Before Next Release)

1. **REVOKE the hardcoded refresh token immediately:**
   ```bash
   # Revoke the token at the auth provider (Keycloak / dh-auth.io)
   curl -X POST https://talabat.dh-auth.io/oauth/revoke \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -d 'token=efj9zpqg7lazmjcuej5tcaavwaxsjkbwrrywf9e6&token_type_hint=refresh_token'
   ```

2. **Force password reset for the exposed test account:**
   - Email: `remixtalabat@gmail.com`
   - Customer ID: `40415224`

### Code Fixes

#### Fix 1: Exclude integration test fixtures from release builds

In `build.gradle` (app-level):
```groovy
android {
    buildTypes {
        release {
            // Exclude integration test fixtures from release APK
            aaptOptions {
                ignoreAssetsPattern 'integration_test/'
            }
        }
    }
}
```

#### Fix 2: Add Gradle task to strip test fixtures

```groovy
tasks.register('stripTestFixtures') {
    doLast {
        fileTree(dir: '${buildDir}/intermediates/assets/release',
                includes: ['**/integration_test/**']).each { it.delete() }
    }
}
tasks.named('mergeReleaseAssets').configure {
    finalizedBy('stripTestFixtures')
}
```

#### Fix 3: Implement device-bound tokens

```kotlin
// Use Android Keystore to bind tokens to device
val keyStore = KeyStore.getInstance("AndroidKeyStore")
keyStore.load(null)

// Store refresh token in EncryptedSharedPreferences
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

val encryptedPrefs = EncryptedSharedPreferences.create(
    context,
    "secure_auth_prefs",
    masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)
```

### Verification

After patching, verify with:
```bash
# 1. Verify fixture is not in release APK
unzip -l app-release.apk | grep 'integration_test'
# Should return empty

# 2. Verify refresh token is revoked
curl -X POST https://talabat.dh-auth.io/oauth/token \
  -d 'grant_type=refresh_token&refresh_token=efj9zpqg7lazmjcuej5tcaavwaxsjkbwrrywf9e6'
# Should return 401 Unauthorized
```

---
## EXP-03: Payment WebView CDN Injection (CRITICAL - CVSS 8.6)

### Immediate Actions

1. **Bundle jQuery locally instead of loading from CDN**
2. **Add Subresource Integrity (SRI) hashes**
3. **Implement Content Security Policy**

### Code Fixes

#### Fix 1: Bundle jQuery locally in assets

Replace in `assets/copyAndPay.html`:

```html
<!-- BEFORE (VULNERABLE) -->
<script src="https://code.jquery.com/jquery.min.js"></script>

<!-- AFTER (FIXED) -->
<script src="file:///android_asset/jquery.min.js"></script>
```

And add `jquery.min.js` to the `assets/` directory in the APK.

#### Fix 2: Add SRI (if CDN must be used)

```html
<script 
  src="https://code.jquery.com/jquery-3.7.1.min.js"
  integrity="sha384-1H217gvSV2JJsrZdkVR9KvKTe1c5S5J3L5d9Ceaub5hVKqBhP7g5j9Rq8fbO3Yv"
  crossorigin="anonymous"
></script>
```

#### Fix 3: Add Content Security Policy

Add CSP meta tag in the `<head>`:

```html
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'none'; 
               script-src 'self' https://code.jquery.com; 
               connect-src {baseUrl}; 
               form-src {shopperResultUrl}; 
               style-src 'self' 'unsafe-inline'; 
               img-src 'self' data:;">
```

#### Fix 4: Sanitize template parameters

```kotlin
// Validate all template parameters before substitution
fun sanitizePaymentTemplate(params: Map<String, String>): Map<String, String> {
    val allowedBaseUrl = Regex("^https://[a-zA-Z0-9.-]+\\.oppwa\\.com(/.*)?$")
    val allowedShopperUrl = Regex("^https://[a-zA-Z0-9.-]+\\.talabat\\.com(/.*)?$")
    
    val sanitized = mutableMapOf<String, String>()
    params.forEach { (key, value) ->
        when (key) {
            "baseUrl" -> {
                if (allowedBaseUrl.matches(value)) sanitized[key] = value
                else throw SecurityException("Invalid baseUrl: \$value")
            }
            "shopperResultUrl" -> {
                if (allowedShopperUrl.matches(value)) sanitized[key] = value
                else throw SecurityException("Invalid shopperResultUrl: \$value")
            }
            else -> sanitized[key] = value
        }
    }
    return sanitized
}
```

### Verification

```bash
# 1. Check no external CDN scripts in payment HTML
grep 'code.jquery.com' assets/copyAndPay.html
# Should return empty

# 2. Verify SRI attribute exists
grep 'integrity=' assets/copyAndPay.html
# Should return the SRI hash line

# 3. Verify CSP is present
grep 'Content-Security-Policy' assets/copyAndPay.html
# Should return the CSP meta tag
```

---
## EXP-02: JWT Token Infrastructure Exposure (HIGH - CVSS 7.5)

### Immediate Actions

1. **Strip test fixtures from production builds** (same as EXP-01 Fix 1)
2. **Migrate from sequential integer customer IDs to UUIDs**
3. **Restrict OIDC discovery endpoint access**

### Code Fixes

#### Fix 1: Use UUIDs instead of sequential customer IDs

```sql
-- Migration: Add UUID column to customers table
ALTER TABLE customers ADD COLUMN uuid UUID DEFAULT gen_random_uuid();

-- Update existing records
UPDATE customers SET uuid = gen_random_uuid() WHERE uuid IS NULL;

-- Make UUID the primary identifier
CREATE UNIQUE INDEX idx_customers_uuid ON customers(uuid);
```

```kotlin
// Update JWT claims to use UUID
data class JwtClaims(
    val iss: String = "https://talabat.dh-auth.io",
    val sub: String,  // Now UUID instead of sequential integer
    val aud: String = "android",
    // ... other claims
)
```

#### Fix 2: Restrict OIDC discovery endpoint

```nginx
# In reverse proxy / API gateway config
location /.well-known/ {
    # Only allow access from known backend IPs
    allow 10.0.0.0/8;
    allow 172.16.0.0/12;
    deny all;
    
    proxy_pass http://keycloak:8080;
}
```

#### Fix 3: Remove email from JWT claims

```kotlin
// BEFORE: Email exposed in JWT payload
// "metadata": {"email": "remixtalabat@gmail.com"}

// AFTER: Remove PII from JWT, fetch via userinfo endpoint
data class JwtClaims(
    val iss: String,
    val sub: String,  // UUID only
    val aud: String,
    val exp: Long,
    val iat: Long,
    val jti: String,
    // No metadata with email
)
```

#### Fix 4: Rotate signing keys

```bash
# Rotate the compromised signing key
# In Keycloak / dh-auth.io admin console:
# 1. Generate new key pair
# 2. Set new key as active
# 3. Disable keymaker-talabat-0026-android
# 4. Keep old key for verification only until all tokens expire
```

### Verification

```bash
# 1. Verify OIDC discovery is restricted
curl https://talabat.dh-auth.io/.well-known/openid-configuration
# Should return 403 Forbidden from external IPs

# 2. Verify customer IDs are UUIDs
# Check JWT payload sub claim - should be UUID format
```

---
## EXP-04: Exported Payment WebView Activity (HIGH - CVSS 7.3)

### Immediate Actions

1. **Set `android:exported="false"` in AndroidManifest.xml**
2. **Add URL allowlist validation**
3. **Add caller package validation**

### Code Fixes

#### Fix 1: Remove exported flag

In `AndroidManifest.xml`:

```xml
<!-- BEFORE (VULNERABLE) -->
<activity
    android:name="com.talabat.secure_payment_redirection.SecurePaymentRedirectionWebViewActivity"
    android:exported="true"
    ...>
</activity>

<!-- AFTER (FIXED) -->
<activity
    android:name="com.talabat.secure_payment_redirection.SecurePaymentRedirectionWebViewActivity"
    android:exported="false"
    ...>
</activity>
```

#### Fix 2: Add URL allowlist validation

In `SecurePaymentRedirectionWebViewActivity.java` / `.kt`:

```kotlin
class SecurePaymentRedirectionWebViewActivity : AppCompatActivity() {

    companion object {
        // Allowlist of permitted domains for payment redirection
        private val ALLOWED_DOMAINS = listOf(
            ".oppwa.com",        // Payment processor
            ".talabat.com",       // Talabat domains
            ".dh-auth.io"         // Auth domain
        )
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        val redirectUrl = intent.getStringExtra("redirectUrl") ?: ""
        
        // Validate URL against allowlist
        if (!isUrlAllowed(redirectUrl)) {
            Log.e(TAG, "Blocked unauthorized URL: \$redirectUrl")
            finish()  // Close activity immediately
            return
        }
        
        // Also validate caller package
        if (!isCallerValid()) {
            Log.e(TAG, "Blocked unauthorized caller")
            finish()
            return
        }
        
        webView.loadUrl(redirectUrl)
    }

    private fun isUrlAllowed(url: String): Boolean {
        if (url.isBlank()) return false
        if (url.startsWith("javascript:")) return false  // Block JS URIs
        if (url.startsWith("file:")) return false       // Block file URIs
        if (url.startsWith("data:")) return false        // Block data URIs
        
        val host = try { URL(url).host } catch (_: Exception) { return false }
        return ALLOWED_DOMAINS.any { domain -> host.endsWith(domain) }
    }

    private fun isCallerValid(): Boolean {
        val callerPackage = callingActivity?.packageName ?: return false
        return callerPackage == "com.talabat" ||
               callerPackage.startsWith("com.talabat.")
    }
}
```

#### Fix 3: Disable JavaScript for non-essential WebView operations

```kotlin
// Only enable JavaScript for specific known payment URLs
val isPaymentProcessor = redirectUrl.contains("oppwa.com")
webView.settings.javaScriptEnabled = isPaymentProcessor
```

#### Fix 4: Add @NonNull annotation and null checks

```kotlin
// Replace the current unsafe code:
// String stringExtra = getIntent().getStringExtra("redirectUrl");
// String str = stringExtra != null ? stringExtra : "";
// webView.loadUrl(str);

// With safe version:
val redirectUrl = intent.getStringExtra("redirectUrl")
    ?.takeIf { isUrlAllowed(it) }
    ?: run {
        Log.w(TAG, "Invalid or missing redirectUrl")
        finish()
        return
    }
webView.loadUrl(redirectUrl)
```

### Verification

```bash
# 1. Verify activity is no longer exported
aapt dump xmltree app-release.apk AndroidManifest.xml | grep -A5 SecurePayment
# Should show android:exported=0xffffffff (false)

# 2. Test that external intent is blocked
adb shell am start -n com.talabat/.secure_payment_redirection.SecurePaymentRedirectionWebViewActivity --es redirectUrl 'https://evil.com'
# Should return: Permission Denial: not exported from uid

# 3. Test URL allowlist
# Internal app calls with valid URLs should still work
# External calls with any URL should be blocked
```

---
## Patch Priority Order

| Priority | Exploit | Action | Timeline |
|----------|---------|--------|----------|
| P0 | EXP-01 | Revoke hardcoded refresh token | IMMEDIATE (today) |
| P0 | EXP-01 | Exclude integration_test from release builds | 24 hours |
| P0 | EXP-03 | Bundle jQuery locally, add SRI + CSP | 48 hours |
| P1 | EXP-04 | Set exported=false, add URL validation | 48 hours |
| P1 | EXP-02 | Migrate to UUID customer IDs | 1-2 weeks |
| P2 | EXP-02 | Restrict OIDC discovery, rotate signing keys | 1 week |
| P2 | EXP-04 | Add caller package validation | 1 week |

## Post-Patch Testing Checklist

- [ ] Run `unzip -l app-release.apk | grep integration_test` - should be empty
- [ ] Run `unzip -l app-release.apk | grep copyAndPay` - should contain only local jQuery
- [ ] Test `adb shell am start` with external redirectUrl - should be blocked
- [ ] Verify refresh token returns 401 from auth endpoint
- [ ] Verify OIDC discovery returns 403 from external IPs
- [ ] Verify JWT sub claim is UUID format, not sequential integer
- [ ] Run full regression test suite for payment flow
