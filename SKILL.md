---
name: japan-recs
description: Research restaurants, activities, and places to visit in a specific area of Japan using forums, reviews, and social media, then compile the results into Google Sheets, CSV, or Markdown.
disable-model-invocation: false
user-invocable: true
allowed-tools: Bash, Agent, WebSearch, WebFetch, Write
argument-hint: [area] [--activities | --both | --vibes] [--csv | --sheet SPREADSHEET_ID]
---

# Japan Travel Research & Recommendation Builder

Research top recommendations for a location in Japan and output the results in your preferred format. Supports both restaurant research and activity/experience research.

## Arguments

```
/japan-recs Shinjuku                              → Restaurants only, Markdown (default)
/japan-recs Shinjuku --activities                  → Activities only
/japan-recs Shinjuku --both                        → Restaurants + Activities
/japan-recs Shinjuku --vibes                       → Aesthetic/trending-focused research
/japan-recs Shinjuku --vibes --sheet SHEET_ID      → Vibes results to Google Sheets
/japan-recs Tokyo --activities --sheet SHEET_ID    → Activities to Google Sheets
/japan-recs Shinjuku --csv                         → CSV file output
/japan-recs Shinjuku --sheet SHEET_ID              → Google Sheets output (requires gws CLI)
```

- **First argument**: Area or neighborhood in Japan (required)
- `--activities`: Research activities and experiences instead of restaurants
- `--both`: Research both restaurants and activities (outputs as separate tabs/files)
- `--vibes`: Aesthetic/trending-focused research — prioritizes atmosphere, design, and trending signals over pure review scores. Uses vibes-specific categories (see Vibes Categories below) and outputs to a separate tab/file (e.g., "[Area] Vibes")
- `--csv`: Output as CSV file
- `--sheet SPREADSHEET_ID`: Output to a Google Sheet tab (requires `gws` CLI to be installed and authenticated)

If no type flag is provided, defaults to restaurants only.
If no output flag is provided, defaults to Markdown.

## Workflow

### Step 1: Research

Spawn **parallel Agents** (one per topic area) to research across multiple sources simultaneously. Each agent should search:
- Reddit (r/JapanTravel, r/Tokyo, r/JapanFood, r/japanlife)
- Google Reviews / TripAdvisor ratings and comments
- Tabelog (Japan's top restaurant review site — for restaurants)
- Food blogs (Ramen Adventures, Food Sake Tokyo, ByFood, etc. — for restaurants)
- TimeOut Tokyo, SAVOR JAPAN, Japan Food Guide
- Viator, Klook (for activities/experiences)
- Social media recommendations

#### For restaurant research, gather per place:
- **Name** (English + Japanese if available)
- **Category** (Ramen, Sushi, Yakiniku, Izakaya, Soba, Tonkatsu, Teppanyaki, Fine Dining, Desserts & Sweets, Cafe, etc.)
- **What it's known for** (signature dish or experience)
- **Price range** (Budget / Mid-range / Splurge, with yen amounts when available)
- **Vibe** (1-2 tags from: `Classic`, `Hidden Gem`, `Trending`, `Aesthetic`, `Hole-in-the-wall`, `Splurge`, `Late Night` — see Vibe Tags below)
- **Address** (English, with Japanese building name if applicable)
- **Whether reservations are needed**
- **Why people recommend it** (specific praise from forums/reviews — this becomes the Notes column and should be detailed: include specific quotes from reviewers, what makes it stand out, atmosphere, who it's best for, any tips like best time to go or what to order)
- **Google Maps link** (search for the restaurant name + area to construct a Maps URL)
- **Reservation/review link** (Tabelog, TripAdvisor, or official site URL — whichever is most useful for booking or reading reviews)

Prioritize places that appear across multiple sources. Aim for 15-25 recommendations with a mix of budget, mid-range, and splurge options across different food categories.

**Trending signals**: In ALL modes, research agents should also look for curated proxy signals that indicate a place is currently buzzy or newly popular:
- **TimeOut Tokyo** "best new restaurants/bars/cafes [year]" lists
- **Tabelog** "new entry" awards, The Tabelog Award Bronze/Silver/Gold new additions
- **Michelin** new Bib Gourmand and starred entries for the current year
- **Reddit** posts with language like "just discovered", "new favorite", "recently opened", "hidden gem I found"
- **Design press** coverage (Wallpaper*, Dezeen, Monocle, ArchDaily) — a place featured in design press is almost always aesthetic + trending
- **Food media** new features (Eater, Bon Appétit, Bloomberg Pursuits Asia features)

These signals help surface trending places alongside established classics. In `--vibes` mode, weight these signals more heavily.

#### For activity research, gather per place:
- **Name** (English + Japanese if available)
- **Category** (see Activity Categories below)
- **What it's known for** (1-line summary of the experience)
- **Price range** (entry fees, typical spend, with yen amounts)
- **Vibe** (1-2 tags from: `Classic`, `Hidden Gem`, `Trending`, `Aesthetic`, `Hole-in-the-wall`, `Splurge`, `Late Night` — see Vibe Tags below)
- **Address** (full address or area if it's a district/neighborhood)
- **Whether reservations/tickets are needed** (and how far in advance)
- **Detailed notes** (what Reddit/forums say, tips, best time to visit, how long to spend, group-friendliness, seasonal relevance)
- **Google Maps link** (constructed search URL)
- **Review/website link** (TripAdvisor, official site, or booking platform)

**Parallelization strategy for activities**: Spawn separate agents for each topic cluster to maximize research speed:
1. **Nightlife agent**: Clubs, themed bars, karaoke
2. **Cultural agent**: Temples, shrines, museums, parks/gardens
3. **Adventure agent**: Gaming/arcades, unique experiences (go-karts, onsen, sumo), sports/outdoors
4. **Shopping agent**: Shopping districts, markets, specialty stores

Aim for 4-6 places per sub-category, 50-70 total activities.

### Step 2: Get Addresses

If addresses weren't found in Step 1, spawn a second Agent to look up addresses for all places found. Include the building name in Japanese when available (e.g., "1-1-7 Ebisu, Shibuya-ku (117ビル 1F)").

### Step 3: Output Results

Output the results based on the user's chosen format:

---

#### Option A: Markdown (default)

**Restaurants** → Write to `./japan-recs/[Area] Restaurants.md`
**Activities** → Write to `./japan-recs/[Area] Activities.md`

Format the file as:

```markdown
# [AREA NAME] Restaurant Recommendations

## 🍜 Ramen

| Name | Known For | Price | Vibe | Address | Maps | Link | Reservations | Status | Notes |
|------|-----------|-------|------|---------|------|------|--------------|--------|-------|
| Restaurant Name | Signature dish | ¥1,000 | Hidden Gem | Address | [Maps](url) | [Tabelog](url) | No | | Detailed notes... |

## 🍣 Sushi

| Name | Known For | Price | Vibe | Address | Maps | Link | Reservations | Status | Notes |
|------|-----------|-------|------|---------|------|------|--------------|--------|-------|
...
```

For activities, use the same table structure but with the activity category headers and `Review/Website Link` instead of `Reservation/Review Link`.

Leave the **Status** column empty — the user uses this to track bookings.

---

#### Option B: CSV (`--csv`)

**Restaurants** → Write to `./japan-recs/[Area] Restaurants.csv`
**Activities** → Write to `./japan-recs/[Area] Activities.csv`

Columns: `Category,Name,Known For,Price,Vibe,Address,Google Maps,Review/Website Link,Reservations,Status,Notes`

- Include the category emoji prefix in the Category column (e.g., "🍜 Ramen" or "🎮 Gaming & Arcades")
- Leave the Status column empty
- Properly escape any commas or quotes in field values

---

#### Option C: Google Sheets (`--sheet SPREADSHEET_ID`)

Use the `gws` CLI to write results to the provided spreadsheet.

1. Create a new tab named after the area (e.g., "Shinjuku Restaurants" or "Tokyo Activities")
2. Clear any existing data if the tab already exists
3. Write data in this format:

```
Row 1: [AREA NAME] RESTAURANT RECOMMENDATIONS  (or ACTIVITIES & EXPERIENCES)
Row 2: (blank)
Row 3: [Category emoji] CATEGORY NAME
Row 4: Name | Known For | Price | Vibe | Address | Google Maps | Review/Website Link | Reservations | Status | Notes
Row 5+: Data rows
(blank row)
Row N: [Next category emoji] NEXT CATEGORY
Row N+1: Name | Known For | Price | Vibe | Address | Google Maps | Review/Website Link | Reservations | Status | Notes
...
```

Leave the **Status** column empty.

**For large datasets (50+ entries)**: Build the complete data array in a Python script, write to a temp JSON file, then pass it to gws in a single `values update` call. This is more reliable than constructing huge JSON strings inline in bash.

---

### Restaurant Category Emojis

Ramen 🍜, Yakiniku 🥩, Sushi 🍣, Izakaya/Bars 🍺, Tonkatsu/Soba/Teishoku 🥢, Teppanyaki/Fine Dining 🔥, Desserts & Sweets 🍡, Cafe ☕, Other 🍽️

**Always include a Desserts & Sweets section** (🍡) — research dessert spots, bakeries, mochi shops, kakigori, parfaits, taiyaki, crepes, matcha desserts, etc. specific to the area.

### Activity Category Emojis

| Emoji | Category | What to include |
|-------|----------|-----------------|
| 🎪 | Clubs & Nightlife | Nightclubs, dance venues, live music |
| 🍸 | Unique Nightlife / Themed | Themed bars, comedy izakayas, bar districts (Golden Gai, Omoide Yokocho), quirky venues |
| 🎤 | Karaoke | Karaoke chains and premium options with party plans |
| ⛩️ | Temples & Shrines | Major and hidden-gem temples and shrines |
| 🎨 | Museums & Galleries | Art museums, cultural museums, immersive experiences (teamLab) |
| 🌸 | Parks & Cherry Blossoms | Parks, gardens, seasonal viewing spots (cherry blossoms, autumn leaves) |
| 🎮 | Gaming & Arcades | Arcades, retro game shops, VR experiences, entertainment complexes |
| 🎯 | Unique Experiences | Go-karts, onsen/sento, sumo, sword experiences, market tours, capsule hotels, batting cages |
| ⛰️ | Sports & Outdoors | Hiking, day trips, baseball games, cycling tours |
| 🛍️ | Shopping Districts & Markets | Shopping neighborhoods, markets, specialty stores (knives, stationery, whisky) |

**Adapt categories to the area**: Not all categories apply everywhere. A research query for "Shinjuku" might emphasize nightlife and karaoke, while "Kamakura" would focus on temples and outdoors. Include what's relevant.

**Seasonal awareness**: If the user mentions a travel month, tailor recommendations accordingly (cherry blossoms in March-April, autumn leaves in November, festivals, etc.).

### Vibe Tags

Every place in every mode (restaurants, activities, vibes) gets tagged with 1-2 vibe descriptors in the `Vibe` column:

| Tag | Meaning |
|-----|---------|
| `Classic` | Established institution, decades of history |
| `Hidden Gem` | Under-the-radar, locals-only, hard to find |
| `Trending` | Recently opened or currently buzzy on social media |
| `Aesthetic` | Design-forward, beautiful interior, photogenic |
| `Hole-in-the-wall` | No-frills, tiny, character-driven |
| `Splurge` | Premium experience, worth the price for the occasion |
| `Late Night` | Open late, good for after-hours |

Tags can be combined (e.g., "Trending, Aesthetic" or "Classic, Hole-in-the-wall"). Choose the 1-2 most defining tags for each place.

### Vibes Categories (used when `--vibes` flag is set)

When `--vibes` is set, use these categories instead of the standard restaurant or activity categories:

| Emoji | Category |
|-------|----------|
| ☕ | Aesthetic Cafes |
| 🍸 | Aesthetic Bars & Cocktail Lounges |
| 🍽️ | Aesthetic Restaurants |
| 🏙️ | Aesthetic Neighborhoods & Districts |
| 🏪 | Food Halls & Multi-Vendor Spaces |

**Vibes file/tab naming**: `[Area] Vibes` (e.g., "Shinjuku Vibes")

**Vibes-mode research agent strategy**: Spawn 4 parallel agents with vibes-tuned prompts:

1. **Aesthetic cafes agent**: Search design publications, "most beautiful cafe" lists, architect-designed spaces, kissaten with character, specialty coffee with standout interiors. Prioritize places featured in Wallpaper*, Dezeen, Monocle, ArchDaily, and Instagram "best cafes in [area]" lists.

2. **Aesthetic bars agent**: Search for speakeasies, hidden bars, rooftop views, World's 50 Best Bars entries, design-forward cocktail lounges, jazz bars with atmosphere. Look at TimeOut "best bars", cocktail competition winners, and design press features.

3. **Aesthetic restaurants agent**: Search for stunning interiors, unique settings (converted warehouses, garden dining, riverside), food-as-art presentation, Michelin-starred spots with design appeal. Check Wallpaper* City Guides, Monocle travel features, and "most beautiful restaurants" lists.

4. **Neighborhood vibes agent**: Search for area guides describing the *feel* of walking through specific neighborhoods, curated food crawl routes, food halls and multi-vendor spaces (depachika, yokocho, etc.), and "best neighborhoods for [eating/drinking/exploring]" guides.

In vibes mode, weight trending signals and design press mentions more heavily than pure review scores. A place with a 3.8 Google rating but featured in Wallpaper* is a better vibes pick than a 4.5-rated institution with no design appeal.

### Links

For each restaurant, include two links:

1. **Google Maps** — construct as: `https://www.google.com/maps/search/?api=1&query=RESTAURANT+NAME+AREA+Tokyo`
   - URL-encode the restaurant name (spaces become `+`)
   - Example: `https://www.google.com/maps/search/?api=1&query=AFURI+Ebisu+Tokyo`

2. **Reservation/Review link** — use the best available from research:
   - Tabelog page (preferred for Japanese restaurants)
   - TripAdvisor page
   - Official website
   - Booking platform (TableCheck, OMAKASE, Klook)
   - If no specific link was found during research, use a Google search URL: `https://www.google.com/search?q=RESTAURANT+NAME+AREA+Tokyo+reservations`

## GWS CLI Reference (for Google Sheets mode)

### Shell escaping
Always pass params via a variable to avoid `!` escaping issues:
```bash
PARAMS='{"spreadsheetId":"SHEET_ID","range":"TabName!A1","valueInputOption":"RAW"}'
gws sheets spreadsheets values update --params "$PARAMS" --json '{"values":[...]}'
```

### Create a new tab
```bash
PARAMS='{"spreadsheetId":"SHEET_ID"}'
gws sheets spreadsheets batchUpdate --params "$PARAMS" --json '{"requests":[{"addSheet":{"properties":{"title":"Tab Name"}}}]}'
```

### Write data
```bash
PARAMS='{"spreadsheetId":"SHEET_ID","range":"Tab Name!A1","valueInputOption":"RAW"}'
gws sheets spreadsheets values update --params "$PARAMS" --json '{"values":[["col1","col2"],["col1","col2"]]}'
```

### Clear data
```bash
PARAMS='{"spreadsheetId":"SHEET_ID","range":"Tab Name!A1:Z1000"}'
gws sheets spreadsheets values clear --params "$PARAMS"
```

### Read data
```bash
PARAMS='{"spreadsheetId":"SHEET_ID","range":"Tab Name"}'
gws sheets spreadsheets values get --params "$PARAMS"
```
