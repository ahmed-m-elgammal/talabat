# 07 — Search & Discovery MVP

## Overview

The MVP search and discovery system provides **basic full-text search** and **category-based browsing** for finding restaurants and dishes. The full Talabat architecture (see `docs/16_search_discovery_system.md`) uses Elasticsearch with Arabic/English tokenization, personalized recommendations, sponsored content via AdTech SDK, AI-powered conversational discovery (ChatGPT), multi-search for grocery lists, and photo-to-list image recognition — supporting 9 countries with complex multilingual needs.

The MVP strips this down to **PostgreSQL full-text search** with `tsvector`/`tsquery`, simple category browsing, and recent-search persistence. This is sufficient for 50–100 vendors with a single language (English), serving 100–500 daily searches. Arabic tokenization, Elasticsearch, personalization, ads, and AI features are all deferred to Phase 2.

---

## MVP Tasks

### T7.1 — Vendor Search API (PostgreSQL Full-Text)

**Description:** Implements the core search endpoint using PostgreSQL `tsvector` for full-text matching against vendor names, cuisine types, and menu item names. The search returns vendors ranked by a composite of text relevance and rating. For MVP, only English text is supported; Arabic tokenization and transliteration are deferred. The endpoint supports pagination and basic result metadata (total count, has_more flag). Search is scoped to the user's delivery area (vendors within radius).

**Dependencies:** T1.2 (vendor service module for vendor data), T1.7 (database schema with tsvector columns).

**Acceptance Criteria:**
- [ ] `GET /v1/search?q={query}&lat={lat}&lng={lng}&page=1&limit=20` returns vendors matching the query
- [ ] Search matches against: vendor name, cuisine types, and menu item names (joined)
- [ ] Results ranked by: text relevance (ts_rank) + vendor rating (weighted 70/30)
- [ ] Only vendors within delivery radius of the user's location are returned
- [ ] Only open vendors appear in results by default (closed vendors can be shown with `?include_closed=true`)
- [ ] Pagination: `page` and `limit` parameters; response includes `total_count` and `has_more`
- [ ] Empty query (`q=`) returns top-rated nearby vendors (fallback to browsing)
- [ ] Query is trimmed and lowercased; empty-after-trim returns 400 with clear message
- [ ] Response includes: vendor_id, name, cuisine_types, rating_avg, delivery_time_min/max, delivery_fee, is_open, thumbnail_url
- [ ] Search latency target: < 500ms for 50–100 vendors

**Phase:** MVP

---

### T7.2 — Menu Item Search (Within Vendor)

**Description:** Allows searching for specific dishes/items within a vendor's menu. This powers the "search this restaurant" feature on the vendor detail screen. Uses the same PostgreSQL `tsvector` approach scoped to a single vendor's menu items. Results are grouped by menu category.

**Dependencies:** T7.1 (search API infrastructure), T1.2 (vendor/menu data).

**Acceptance Criteria:**
- [ ] `GET /v1/vendors/{id}/menu/search?q={query}` returns matching menu items
- [ ] Search matches against: item name and item description
- [ ] Results grouped by menu category with category name and matching items
- [ ] Unavailable items (is_available = false) are shown but greyed out with "Unavailable" label
- [ ] Empty query returns the full menu (same as GET /vendors/{id}/menu)
- [ ] No matches returns empty array with 200 status (not 404)
- [ ] Response includes: item_id, name, description, base_price, is_available, category_name

**Phase:** MVP

---

### T7.3 — Category Browsing

**Description:** Implements category-based browsing as the primary discovery mechanism alongside search. Customers can browse vendors by cuisine type (e.g., "Burger", "Pizza", "Indian", "Arabic"). Categories are predefined for MVP (no dynamic category management). Selecting a category shows vendors filtered by cuisine type, sorted by rating and distance.

**Dependencies:** T1.2 (vendor service with cuisine type data).

**Acceptance Criteria:**
- [ ] `GET /v1/categories` returns list of cuisine categories with icon URLs and vendor count
- [ ] Predefined MVP categories: Burger, Pizza, Indian, Arabic, Italian, Asian, Desserts, Healthy, Breakfast, Shawarma
- [ ] Each category includes: id, name, icon_url, vendor_count (within user's delivery area)
- [ ] `GET /v1/categories/{id}/vendors?lat={lat}&lng={lng}&page=1&limit=20` returns vendors in that category
- [ ] Vendors sorted by: rating (descending), then distance (ascending) as tiebreaker
- [ ] Only open vendors shown by default; closed vendors accessible with `?include_closed=true`
- [ ] Category vendor count updates when vendors open/close (reflected on next request or cache refresh)
- [ ] Response format identical to vendor listing API for consistent frontend rendering

**Phase:** MVP

---

### T7.4 — Recent Searches (Client-Side)

**Description:** Persists the user's recent search queries locally on the device using SQLite (Flutter's `sqflite` package). Shows recent searches when the search bar is tapped, before any query is entered. This provides a lightweight "discovery" experience without server-side infrastructure. Recent searches are stored with a timestamp and automatically pruned after 7 days.

**Dependencies:** T2.3 (search bar UI component).

**Acceptance Criteria:**
- [ ] When search bar is tapped (no query entered), show up to 10 recent searches
- [ ] Each recent search shows query text and timestamp (e.g., "2 hours ago", "Yesterday")
- [ ] Tapping a recent search executes that query immediately
- [ ] "Clear recent searches" option removes all stored searches
- [ ] Individual searches can be removed by swipe-to-delete
- [ ] Searches stored in local SQLite: `{query_text, searched_at}`
- [ ] Duplicate queries move to top (not duplicated in list)
- [ ] Entries older than 7 days are automatically pruned on app launch
- [ ] Recent searches persist across app restarts

**Phase:** MVP

---

### T7.5 — Search Autocomplete (Simplified)

**Description:** Provides basic autocomplete suggestions as the user types in the search bar. For MVP, autocomplete is powered by PostgreSQL prefix matching against vendor names and a curated list of popular search terms (no Elasticsearch n-gram tokenizer). Suggestions appear after the user types at least 2 characters, with a 300ms debounce to avoid excessive API calls.

**Dependencies:** T7.1 (search API), T2.3 (search bar UI).

**Acceptance Criteria:**
- [ ] `GET /v1/search/suggest?q={prefix}` returns up to 8 suggestions
- [ ] Suggestions sourced from: vendor names (prefix match) + popular search terms (curated list in DB)
- [ ] Minimum 2 characters required; 1-character queries return empty suggestions
- [ ] 300ms debounce on the client side (no API call until user stops typing for 300ms)
- [ ] Results include: suggestion text, type ("vendor" | "query"), and vendor_id (if type is vendor)
- [ ] Tapping a vendor suggestion navigates to vendor detail screen
- [ ] Tapping a query suggestion executes a full search with that query
- [ ] Popular search terms are seeded with 20–30 common queries (e.g., "burger", "pizza", "shawarma")
- [ ] Suggestion API latency target: < 200ms

**Phase:** MVP

---

### T7.6 — Search Results Screen (UI)

**Description:** Implements the search results screen showing matched vendors in a scrollable list. For MVP, results are displayed in a single flat list (no vertical tabs like Food/Grocery/Health since MVP is food-only). Each result card shows vendor name, cuisine tags, rating, delivery time, delivery fee, and open/closed status. Supports pull-to-refresh and infinite scroll pagination.

**Dependencies:** T7.1 (search API), T2.3 (search bar component).

**Acceptance Criteria:**
- [ ] Search results display as a vertical list of vendor cards
- [ ] Each card shows: vendor name, cuisine tags (max 3), rating with count, delivery time range, delivery fee, open/closed badge
- [ ] Closed vendors shown with reduced opacity and "Closed" badge; tapping shows "Preorder" option
- [ ] Pull-to-refresh re-executes the search query
- [ ] Infinite scroll: next page loads automatically when user scrolls near bottom
- [ ] Loading state: skeleton cards while fetching results
- [ ] Empty state: "No restaurants found for '{query}'" with suggestion to try different keywords
- [ ] Error state: "Something went wrong" with retry button
- [ ] Tapping a vendor card navigates to vendor detail screen
- [ ] Search query persists in search bar; user can modify and re-search
- [ ] Back button returns to home screen (not previous search)

**Phase:** MVP

---

### T7.7 — Sort & Filter (Basic)

**Description:** Adds basic sort and filter controls to the search results screen. For MVP, supports sort by: Recommended (default), Rating, Delivery Time, Distance. Supports filters: Cuisine type (multi-select), Open Now (toggle), Free Delivery (toggle), Min Order Value (range slider). Filters are applied server-side; the filter state is reflected in the URL query parameters.

**Dependencies:** T7.1 (search API with filter/sort parameters), T7.6 (search results screen).

**Acceptance Criteria:**
- [ ] Sort options: Recommended (default), Rating (high to low), Delivery Time (low to high), Distance (near to far)
- [ ] Filter options: Cuisine (multi-select checkboxes), Open Now (toggle), Free Delivery (toggle), Min Order Value (slider: 0–50)
- [ ] Active filters shown as chips below the search bar; individual chips can be removed
- [ ] "Clear All" button removes all active filters
- [ ] Sort and filter panel accessible via a bottom sheet (triggered by "Sort & Filter" button)
- [ ] Applying filters/sort executes a new search with updated parameters
- [ ] Filter state preserved when paginating through results
- [ ] "Filters (N)" badge on filter button shows count of active filters
- [ ] API supports: `?sort=rating&cuisines=indian,arabic&open_now=true&free_delivery=true&max_min_order=25`

**Phase:** MVP

---

### T7.8 — Home Screen Discovery Sections

**Description:** Implements the discovery sections on the home screen that appear before any search is initiated. These sections help customers browse without needing to know what they want. MVP sections include: "Popular near you" (top-rated vendors by proximity), "Cuisine categories" (horizontal scroll of category chips), and "New on Talabat" (recently added vendors). These are powered by simple database queries, not a recommendation engine.

**Dependencies:** T1.2 (vendor listing API), T7.3 (category data).

**Acceptance Criteria:**
- [ ] "Popular near you" section: top 10 vendors by rating within delivery radius, displayed as horizontal scrollable cards
- [ ] "Cuisine categories" section: horizontal scroll of category chips with icons (links to category browsing)
- [ ] "New on Talabat" section: up to 5 vendors created in the last 30 days, shown as vertical list
- [ ] Each section has a "See all" link that navigates to the full listing with appropriate filters
- [ ] Sections load in parallel (not sequentially) for faster page render
- [ ] Vendor cards are identical in format to search result cards for consistency
- [ ] Data cached for 5 minutes (same TTL as vendor listing cache)
- [ ] Empty sections are hidden (e.g., no new vendors in last 30 days → hide "New on Talabat")

**Phase:** MVP

---

### T7.9 — Database Setup for Search (tsvector & Indexes)

**Description:** Adds PostgreSQL tsvector columns and GIN indexes to support full-text search on vendors and menu items. This includes a migration to add a `search_vector` column to both the `vendors` and `menu_items` tables, a trigger to automatically update the tsvector on insert/update, and GIN indexes for fast lookup. Also seeds popular search terms into a `popular_search_terms` table.

**Dependencies:** T1.7 (base database schema).

**Acceptance Criteria:**
- [ ] Migration adds `search_vector tsvector` column to `vendors` table
- [ ] Migration adds `search_vector tsvector` column to `menu_items` table
- [ ] Trigger function `update_vendor_search_vector()` concatenates vendor name + cuisine types into tsvector
- [ ] Trigger function `update_menu_item_search_vector()` concatenates item name + description into tsvector
- [ ] Triggers fire on INSERT and UPDATE for both tables
- [ ] GIN index on `vendors.search_vector` for fast full-text queries
- [ ] GIN index on `menu_items.search_vector` for fast full-text queries
- [ ] Migration also creates `popular_search_terms` table: `{id, term, search_count, language}`
- [ ] Seed script populates `popular_search_terms` with 20–30 common food queries
- [ ] Existing vendor and menu item data is backfilled with tsvector values

**Phase:** MVP

---

## Phase 2 Search & Discovery Items (Not for MVP)

### T7.P2.1 — Elasticsearch Migration
**Description:** Replace PostgreSQL tsvector with Elasticsearch for scalable full-text search. Adds Arabic/English analyzers with diacritics stripping, letter normalization (أ→ا), transliteration support (برجر↔burger), and n-gram tokenizer for autocomplete. Per-country indices with 3 shards + 1 replica. Target search latency: <200ms p95.
**Phase:** Phase 2

### T7.P2.2 — Arabic Language Support
**Description:** Full Arabic language handling: Arabic analyzer with stemmer and stop words, bidirectional text rendering in search UI, dialect-specific synonyms (feature flag `ff_qcommerce_dialects`), Kurdish Sorani analyzer (`ckb.json`), and right-to-left keyboard handling.
**Phase:** Phase 2

### T7.P2.3 — Personalized Recommendation Engine
**Description:** Weighted recommendation system: order history (40%), location proximity (20%), time-of-day patterns (15%), collaborative filtering (15%), browsing behavior (10%). Placements: home "Recommended", search "Recommended restaurants", "Order again", "Deals for you".
**Phase:** Phase 2

### T7.P2.4 — Sponsored Content & AdTech Integration
**Description:** DH AdTech SDK integration for search slot ads, video ads on vendor cards, display ads on home screen. A/B holdout groups for measurement. Animated search bar placeholder with brand suggestions. Sponsored results marked with "Ad" label.
**Phase:** Phase 2

### T7.P2.5 — AI-Powered Discovery (ChatGPT)
**Description:** OpenAI ChatGPT integration for conversational food discovery. Users ask natural language queries ("I want something spicy"), AI recommends vendors/items based on context (location, history, preferences). Includes data transfer disclaimer and "Powered by Chat-GPT API" footer.
**Phase:** Phase 2

### T7.P2.6 — Multi-Search (Q-Commerce Shopping List)
**Description:** Multi-item search: user adds multiple items to a shopping list, API (`POST /search/multi`) finds stores that have ALL items. Enables grocery shopping list experience. Requires feature flag `exp_qcommerce_multi_search`.
**Phase:** Phase 2

### T7.P2.7 — Photo-to-List (Image Recognition)
**Description:** Camera-based shopping list creation: user takes photo of items (e.g., inside refrigerator), AI identifies products, auto-populates shopping list, then executes multi-search. Requires image recognition model and feature flag `exp_qcommerce_photo_to_list`.
**Phase:** Phase 2

### T7.P2.8 — Vertical-Specific Search (Grocery, Pharmacy, Flowers, DineOut)
**Description:** Separate search verticals with custom ranking: Grocery (product availability + store proximity), Pharmacy (medication names, prescription-gated items), Flowers (occasion-based filtering), DineOut (amenity filters: outdoor seating, valet parking). Combined results view with vertical tabs.
**Phase:** Phase 2

### T7.P2.9 — Search Analytics & Quality Monitoring
**Description:** Perseus analytics pipeline for search: track `search_performed`, `search_result_clicked`, `search_zero_results`, `filter_applied`, `search_abandoned` events. Real-time monitoring of latency (p50/p95/p99), zero-result rate (<5%), CTR (>30%), conversion rate (>10%), NDCG relevance (>0.8).
**Phase:** Phase 2

### T7.P2.10 — Search Caching & Offline Support
**Description:** Multi-layer caching: client-side SQLite (5min TTL), Redis server-side (5min), with graceful degradation when offline. Cached response handling with toast messages and "Some restaurants aren't loading" section. Feature flags `exp_search_food_results_cache` for controlled rollout.
**Phase:** Phase 2
