# 16 — Search & Discovery System

## 1. Overview

Talabat's Search & Discovery System is the primary mechanism through which customers find restaurants, stores, dishes, and products across five verticals (Food, Grocery, Pharmacy, Flowers, Speciality Stores). The system combines **full-text search** (powered by Elasticsearch), **personalized recommendations** (based on order history and behavioral signals), **sponsored content** (via the DH AdTech SDK), and **geo-aware ranking** (prioritizing nearby, available vendors). Supporting 9 countries with diverse linguistic needs (Arabic dialects, English, Kurdish), the search system must handle complex multilingual queries, Arabic-specific tokenization challenges (diacritics, transliteration), and real-time inventory filtering.

The discovery layer extends beyond traditional search to include **browsing** (category-based exploration), **curated collections** (editorial content, trending items), and **AI-powered suggestions** (ChatGPT integration for conversational food discovery). The system is optimized for speed (target: < 200ms search latency) and relevance (maximizing order conversion from search results).

---

## 2. Search Architecture

### 2.1 System Architecture

```
Customer App
        │
        ├── Search bar input
        ├── Category browsing
        ├── Home screen recommendations
        └── AI chat queries
        │
        ▼
API Gateway → Search Service
        │
        ├── Query preprocessing
        │   ├── Language detection (Arabic/English/Kurdish)
        │   ├── Tokenization (Arabic-specific: remove diacritics, normalize alef/ya)
        │   ├── Transliteration (Arabic → English, English → Arabic)
        │   ├── Spell correction
        │   └── Synonym expansion (menu item aliases)
        │
        ├── Elasticsearch query
        │   ├── Full-text search (multi-field, boosted)
        │   ├── Geospatial filtering (within delivery radius)
        │   ├── Availability filtering (open, not busy)
        │   ├── Inventory filtering (items in stock)
        │   ├── Personalization boost (based on user history)
        │   ├── Sponsored content injection (AdTech SDK)
        │   └── Facet computation (cuisines, price ranges, delivery times)
        │
        ├── Result ranking
        │   ├── Relevance score (text match + field boost)
        │   ├── Distance score (closer = higher)
        │   ├── Popularity score (order count + rating)
        │   ├── Freshness score (recently ordered by similar users)
        │   ├── Personalization score (user's past preferences)
        │   └── Commercial score (sponsored placement)
        │
        └── Response assembly
            ├── Vendor results (with availability, offers, Pro status)
            ├── Item/dish results (with pricing, images)
            ├── Autocomplete suggestions
            └── Cached results (if API unavailable)
```

### 2.2 Elasticsearch Configuration

**Index Structure:**

```
vendor_index (per country)
├── Mappings:
│   ├── vendor_id (keyword)
│   ├── name (text, analyzer: arabic + english)
│   ├── name_ar (text, analyzer: arabic)
│   ├── cuisine_types (keyword, multi-value)
│   ├── area_names (keyword, multi-value)
│   ├── location (geo_point)
│   ├── rating_avg (float)
│   ├── rating_count (integer)
│   ├── delivery_time_min (integer)
│   ├── delivery_time_max (integer)
│   ├── delivery_fee (float)
│   ├── minimum_order_value (float)
│   ├── is_open (boolean)
│   ├── is_busy (boolean)
│   ├── is_pro_free_delivery (boolean)
│   ├── vertical_type (keyword)
│   ├── menu_items (nested)
│   │   ├── item_id (keyword)
│   │   ├── item_name (text, analyzer: arabic + english)
│   │   ├── item_name_ar (text, analyzer: arabic)
│   │   ├── item_description (text)
│   │   ├── base_price (float)
│   │   ├── is_available (boolean)
│   │   └── cuisine_tags (keyword, multi-value)
│   ├── offers (nested)
│   ├── search_keywords (keyword, multi-value)
│   └── popularity_score (float, decay function)
│
├── Settings:
│   ├── Number of shards: 3 per country
│   ├── Number of replicas: 1
│   ├── Refresh interval: 5s
│   └── Analysis:
│       ├── Arabic analyzer (with stemmer, stop words)
│       ├── English analyzer (standard + stemmer)
│       ├── Custom transliteration filter
│       └── N-gram tokenizer for autocomplete
```

### 2.3 Arabic Language Handling

Arabic search presents unique challenges that the system addresses:

| Challenge | Solution |
|-----------|----------|
| Diacritics (tashkeel) | Strip diacritics before indexing and querying |
| Letter normalization | Normalize أ/إ/آ → ا, ة → ه, ي/ى → ي |
| Transliteration | Support both "burger" and "برجر" for the same query |
| Right-to-left | Bidirectional text handling in search UI |
| Dialects | Feature flag `ff_qcommerce_dialects` enables dialect-specific synonyms |
| Kurdish Sorani | Separate analyzer for `ckb.json` language support |

---

## 3. Search Features

### 3.1 Search Bar

The search bar is context-aware, changing placeholder text based on the current vertical:

| Vertical | Placeholder |
|----------|------------|
| All (Home) | `"search.mvSearchFieldPlaceholder"` = "Search food, groceries and more" |
| Food | `"search.foodListSearchFieldPlaceholder"` = "Search for restaurants or dishes" |
| Grocery | `"search.nfvListSearchFieldPlaceholder"` = "Search for products or stores" |
| General | `"search.searchFieldPlaceholder"` = "Search for food or groceries" |

### 3.2 Autocomplete

As the user types, autocomplete suggestions appear in real-time:

```
┌──────────────────────────────────────────┐
│ 🔍 Search for food or groceries          │
├──────────────────────────────────────────┤
│  🔍 Search for "bur"                     │
│                                          │
│  Recent Searches                         │
│  🕐 burger king                          │
│  🕐 butter chicken                       │
│                                          │
│  Suggestions                             │
│  🔍 burger                               │
│  🔍 burger meal                          │
│  🔍 burrito                              │
│  🔍 Search for "bur" in Food             │
│  🔍 Search for "bur" in Groceries        │
└──────────────────────────────────────────┘
```

**Autocomplete Sources:**
- Recent searches (stored locally in SQLite)
- Popular searches (cached from API, 15-minute TTL in Redis)
- Vendor names (Elasticsearch prefix query)
- Item names (Elasticsearch prefix query)
- Category suggestions

### 3.3 Search Results

Results are displayed in a combined view with vertical-specific sections:

```
┌──────────────────────────────────────────┐
│ 🔍 burger                    ✕ Clear     │
├──────────────────────────────────────────┤
│  All  │  Food  │  Groceries  │  Health   │
├──────────────────────────────────────────┤
│                                          │
│  ── Restaurants ──────────────────       │
│  ┌──────────────────────────────────┐    │
│  │ 🏷️Ad  Burger King               │    │
│  │ Burger · Fast Food · 25-35 min   │    │
│  │ ★ 4.2 (5.6K) · Free delivery    │    │
│  └──────────────────────────────────┘    │
│  ┌──────────────────────────────────┐    │
│  │  Five Guys                       │    │
│  │ Burger · American · 30-45 min    │    │
│  │ ★ 4.6 (3.2K) · AED 7 delivery   │    │
│  └──────────────────────────────────┘    │
│                                          │
│  ── Dishes ───────────────────────       │
│  ┌────┐  Whopper Meal          AED 35   │
│  │ 📷│  Burger King                      │
│  └────┘                                  │
│  ┌────┐  Smash Burger         AED 42    │
│  │ 📷│  Five Guys                        │
│  └────┘                                  │
│                                          │
│  ── View all results in Food ────        │
│  ── View all results in Groceries ──     │
│  ── View all results in Health ────      │
└──────────────────────────────────────────┘
```

**Translation keys for section headers:**
- `"search.combinedSection.viewMore"` = "View more"
- `"search.combinedSection.viewMoreIn"` = "View all results in {verticalName}"
- `"search.combinedSection.viewMoreInFood"` = "View all results in Food"
- `"search.combinedSection.viewMoreInGroceries"` = "View all results in Groceries"
- `"search.combinedSection.viewMoreInHealth"` = "View all results in Health & wellness"
- `"search.combinedSection.viewMoreInFlower"` = "View all results in Flower"
- `"search.combinedSection.viewMoreShops"` = "View all results in Shops"

### 3.4 Pre-Search Recommendations

Before any search query is entered, the search screen displays personalized recommendations:

| Vertical | Title | Description | CTA |
|----------|-------|-------------|-----|
| Grocery | "Shop for all daily essentials" | "From groceries and fresh products to household supplies." | "Search groceries" |
| Pharmacy | "Find health & wellness stores" | "Search for supplements, personal care products, and more." | "Search health & wellness" |
| Flowers | "Find the perfect gift" | "Order beautiful flowers, bouquets, or plants for every occasion." | "Search flowers" |
| More | "Explore far and wide" | "Search for a range of products at a variety of shops." | "Search more products" |

Restaurant recommendations are also shown: `"search.preSearch.vendorRecommendation.title"` = "Recommended restaurants"

---

## 4. Filtering & Sorting

### 4.1 Filter Options

| Filter | Options | Vertical |
|--------|---------|----------|
| Cuisine | Arabic, Indian, Italian, Japanese, Lebanese, American, etc. | Food |
| Delivery type | Delivery, Pickup | Food, Grocery |
| Price range | Budget, Moderate, Premium | Food |
| Rating | 3+, 4+, 4.5+ | Food |
| Delivery time | < 30 min, < 45 min, < 60 min | All |
| Offers | Has offers, Free delivery | All |
| Pro free delivery | Yes/No | All |
| Open now | Yes/No | All |
| Category | Vertical-specific categories | Grocery, Pharmacy |

### 4.2 Sort Options

| Sort | Description | Default |
|------|-------------|---------|
| Recommended | Composite score (relevance + popularity + distance + personalization) | ✅ |
| Delivery time | Shortest to longest | — |
| Rating | Highest to lowest | — |
| Distance | Nearest to farthest | — |
| Min order value | Lowest to highest | — |

### 4.3 Filter UI

```
┌──────────────────────────────────────────┐
│  Sort by ▼  │  Filters (2) ▼  │  Map 📍  │
├──────────────────────────────────────────┤
│                                          │
│  Active Filters: [Cuisine: Lebanese ✕]   │
│                  [Free delivery ✕]       │
│                                          │
│  ── Sort & Filter ───────────────        │
│  ┌──────────────────────────────────┐    │
│  │ Sort by:                         │    │
│  │ ○ Recommended  ● Delivery time   │    │
│  │ ○ Rating       ○ Distance       │    │
│  │                                  │    │
│  │ Cuisine:                         │    │
│  │ ☑ Lebanese  ☐ Indian  ☐ Italian  │    │
│  │ ☐ Japanese  ☐ American           │    │
│  │                                  │    │
│  │ Delivery:                        │    │
│  │ ☑ Free delivery                  │    │
│  │ ☐ Under 30 min                   │    │
│  │                                  │    │
│  │ [Clear all]        [Apply]       │    │
│  └──────────────────────────────────┘    │
└──────────────────────────────────────────┘
```

---

## 5. Multi-Search (Q-Commerce)

### 5.1 Multi-Item Search

The feature flag `exp_qcommerce_multi_search` enables searching for multiple items simultaneously:

```
┌──────────────────────────────────────────┐
│  🛒 Shopping List                         │
│                                          │
│  ┌──────────────────────────────────┐    │
│  │ 🔍 Add item...         [Done ▼] │    │
│  └──────────────────────────────────┘    │
│                                          │
│  Your list:                              │
│  ✕ Milk                                  │
│  ✕ Bread                                 │
│  ✕ Eggs                                  │
│  ✕ Butter                                │
│                                          │
│  ┌──────────────────────────────────┐    │
│  │       Search all items           │    │
│  └──────────────────────────────────┘    │
│                                          │
│  Results show stores that have ALL items  │
└──────────────────────────────────────────┘
```

**API:** `POST /search/multi` with `queries: ["milk", "bread", "eggs", "butter"]`

### 5.2 Photo-to-List

The feature flag `exp_qcommerce_photo_to_list` enables taking a photo of items (e.g., inside a refrigerator) to automatically create a shopping list using image recognition:

1. User taps camera icon
2. Takes photo of items
3. AI identifies products in the image
4. Products are added to shopping list
5. User confirms/modifies the list
6. Multi-search executes for all identified items

---

## 6. Sponsored Content & Ads

### 6.1 AdTech Integration

The DH AdTech SDK (`com.adtechsdk.dh_adtech_sdk_flutter`) provides sponsored content placement:

| Ad Format | Placement | Feature Flag |
|-----------|-----------|-------------|
| Search slot ads | Top of search results | `exp_adex_monetised_search_slots` |
| Video ads | Vendor cards with video | `exp_adex_video_caching` |
| Display ads | Home screen banners | `ff_adex_enable_display_ads_events` |
| Holdout group | Control group for A/B testing | `exp_adex_holdout`, `exp_adex_holdout_v2` |

### 6.2 Sponsored Search Results

Sponsored vendors appear at the top of search results with an "Ad" label:

```
┌──────────────────────────────────────────┐
│  🏷️ Ad  Burger King                      │
│  Burger · Fast Food · 25-35 min          │
│  ★ 4.2 (5.6K) · Free delivery            │
│  Sponsored · "Free delivery"             │
└──────────────────────────────────────────┘
```

Translation keys:
- `"search.sponsored.freeDelivery"` = "Free delivery"
- `"search.more"` = "More" (for additional sponsored items)

### 6.3 Animated Search Text

The search bar supports animated placeholder text with sponsored brand suggestions:

- "Craving McDonald's?"
- "Try KFC today"
- "Hardee's deals inside"
- "Burger King is calling"

---

## 7. AI-Powered Discovery

### 7.1 talabat AI (ChatGPT Integration)

The app integrates **OpenAI's ChatGPT API** for conversational food discovery:

**Translation keys:**
- `"ai_chat_disclaimer"` — Discloses AI usage and data transfer to US
- `"chat_components_footer_powered_by_text"` — "Powered by Chat-GPT API 🤖"

**AI Chat Flow:**
```
1. User opens AI chat (from home or search)
2. Disclaimer shown about OpenAI data processing
3. User types: "Where is my order?" or "I want something spicy"
4. AI understands context (location, order history, preferences)
5. AI recommends specific vendors/items
6. User can tap recommended items to add to cart
```

**Q-Commerce AI Chat:**
Titled "talabat AI" with the hint "Where is my order?" — suggesting the AI can also handle order status queries, not just food recommendations.

### 7.2 Recommendation Engine

The home screen and search results incorporate personalized recommendations:

| Signal | Weight | Source |
|--------|--------|--------|
| Order history | 40% | Past orders (cuisine, vendor, item frequency) |
| Location | 20% | Current delivery address proximity |
| Time of day | 15% | Lunch vs. dinner vs. late-night patterns |
| Similar users | 15% | Collaborative filtering ("Users like you ordered...") |
| Browsing behavior | 10% | Recently viewed vendors/items |

**Recommendation Placements:**

| Placement | Trigger | Content |
|-----------|---------|---------|
| Home "Recommended" section | App open | Personalized vendor cards |
| Search "Recommended restaurants" | Pre-search | Popular + personalized |
| "You might also like" | Item detail | Similar items |
| "Order again" | Order history | Past vendor/items |
| "Deals for you" | Home | Personalized offers (`ds_deals_for_you` icon) |

---

## 8. Search Caching & Offline

### 8.1 Search Result Caching

| Cache Layer | TTL | Feature Flag |
|------------|-----|-------------|
| Client-side (SQLite) | 5 minutes | `exp_search_food_results_cache` |
| Client-side (new key) | 5 minutes | `exp_search_food_results_cache_new_key` |
| Redis (server-side) | 5 minutes | Default |
| Elasticsearch | Real-time | N/A |

### 8.2 Cached Response Handling

When cached results are served (network unavailable or API error):

```
Toast: "Couldn't load some information. Try again."
        (search.cached.response.toast)

Section: "Some restaurants aren't loading"
         (search.cached.vendors.title)

Subtitle (offline): "Looks like you're offline, but here are some picks 
                     which you can still explore"
                     (search.cached.vendors.sub.title.no.internet)

Subtitle (error): "Something went wrong. Here are some choices 
                   you can still explore"
                   (search.cached.vendors.sub.title.error)
```

### 8.3 Search Result Quality Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Search latency (p95) | < 200ms | Time from query submission to result display |
| Zero-result rate | < 5% | Searches returning no results |
| Click-through rate | > 30% | Results clicked / results displayed |
| Conversion rate | > 10% | Orders placed from search / searches |
| Relevance score | > 0.8 (NDCG) | Relevance of top 10 results |
| Autocomplete latency | < 50ms | Suggestion display after keystroke |

---

## 9. Vertical-Specific Search

### 9.1 Food Search

- **Query types**: Restaurant names, dish names, cuisine types
- **Ranking**: Popularity + rating + distance + delivery time
- **Special**: "Preorder only" badge for closed restaurants (`search.preorder`), "Busy" badge for overwhelmed vendors (`search.busy`)

### 9.2 Grocery Search

- **Query types**: Product names, brand names, category names
- **Ranking**: Product availability + store proximity + in-stock guarantee
- **Special**: High/low precision search with "similar items" fallback, PLP filters (`ff_qcommerce_plp_filters_fe`)
- **Age restriction**: Filtered search for age-restricted items

### 9.3 Pharmacy Search

- **Query types**: Medication names, health categories, symptoms
- **Special**: Prescription-gated items (not searchable without prescription), insurance-covered item indicators

### 9.4 DineOut Search

- **Query types**: Restaurant names, cuisine types, amenities
- **Filters**: Outdoor seating, live music, smoking area, pet friendly, valet parking, group dining
- **Sort**: Discount percentage, distance, rating

### 9.5 Flower Search

- **Query types**: Flower types, occasion, arrangement
- **Special**: Occasion-based filtering (birthday, anniversary, sympathy)

---

## 10. Search Analytics

### 10.1 Tracked Events

| Event | Properties | Purpose |
|-------|-----------|---------|
| `search_performed` | query, vertical, results_count, latency_ms | Search volume and performance |
| `search_result_clicked` | query, position, vendor_id, is_sponsored | Click-through analysis |
| `search_result_ordered` | query, vendor_id, order_value | Search-to-order conversion |
| `search_zero_results` | query, vertical | Query gap identification |
| `autocomplete_shown` | prefix, suggestion_count | Autocomplete effectiveness |
| `autocomplete_selected` | prefix, selected_suggestion, position | Autocomplete CTR |
| `filter_applied` | filter_type, filter_value, results_count | Filter usage analysis |
| `sort_changed` | sort_type | Sort preference analysis |
| `search_abandoned` | query, time_on_results | Search funnel analysis |

### 10.2 Search Quality Monitoring

The Perseus analytics pipeline tracks search quality metrics in real-time:

- **Query success rate**: Percentage of searches that lead to a click or order
- **Search latency distribution**: p50, p95, p99 per country and vertical
- **Zero-result queries**: Logged for synonym expansion and content gap analysis
- **Spelling correction rate**: How often spell-corrected queries succeed vs. original
- **Sponsored CTR**: Click-through rate on sponsored positions vs. organic
