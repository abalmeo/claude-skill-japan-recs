---
name: japan-recs
description: Research restaurants, activities, and places to visit in a specific area of Japan using forums, reviews, and social media, then compile the results into Google Sheets, CSV, or Markdown.
disable-model-invocation: false
user-invocable: true
allowed-tools: Bash, Agent, WebSearch, WebFetch, Write
argument-hint: [area] [--csv | --sheet SPREADSHEET_ID]
---

# Japan Travel Research & Recommendation Builder

Research top recommendations for a location in Japan and output the results in your preferred format.

## Arguments

```
/japan-recs Shinjuku                    → Markdown output (default, no setup required)
/japan-recs Shinjuku --csv              → CSV file output
/japan-recs Shinjuku --sheet SHEET_ID   → Google Sheets output (requires gws CLI)
```

- **First argument**: Area or neighborhood in Japan (required)
- `--csv`: Output as CSV file
- `--sheet SPREADSHEET_ID`: Output to a Google Sheet tab (requires `gws` CLI to be installed and authenticated)

If no output flag is provided, defaults to Markdown.

## Workflow

### Step 1: Research

Spawn an Agent to research across multiple sources. The agent should search for:
- Reddit (r/JapanTravel, r/Tokyo, r/JapanFood, r/japanlife)
- Google Reviews / TripAdvisor ratings and comments
- Tabelog (Japan's top restaurant review site)
- Food blogs (Ramen Adventures, Food Sake Tokyo, ByFood, etc.)
- TimeOut Tokyo, SAVOR JAPAN, Japan Food Guide
- Social media recommendations

For each place found, gather:
- **Name** (English + Japanese if available)
- **Category** (Ramen, Sushi, Yakiniku, Izakaya, Soba, Tonkatsu, Teppanyaki, Fine Dining, Desserts & Sweets, Cafe, etc.)
- **What it's known for** (signature dish or experience)
- **Price range** (Budget / Mid-range / Splurge, with yen amounts when available)
- **Address** (English, with Japanese building name if applicable)
- **Whether reservations are needed**
- **Why people recommend it** (specific praise from forums/reviews — this becomes the Notes column and should be detailed: include specific quotes from reviewers, what makes it stand out, atmosphere, who it's best for, any tips like best time to go or what to order)
- **Google Maps link** (search for the restaurant name + area to construct a Maps URL)
- **Reservation/review link** (Tabelog, TripAdvisor, or official site URL — whichever is most useful for booking or reading reviews)

Prioritize places that appear across multiple sources. Aim for 15-25 recommendations with a mix of budget, mid-range, and splurge options across different food categories.

### Step 2: Get Addresses

If addresses weren't found in Step 1, spawn a second Agent to look up addresses for all places found. Include the building name in Japanese when available (e.g., "1-1-7 Ebisu, Shibuya-ku (117ビル 1F)").

### Step 3: Output Results

Output the results based on the user's chosen format:

---

#### Option A: Markdown (default)

Write to `./japan-recs/[Area] Restaurants.md` in the current working directory.

Format the file as:

```markdown
# [AREA NAME] Restaurant Recommendations

## 🍜 Ramen

| Name | Known For | Price | Address | Maps | Link | Reservations | Status | Notes |
|------|-----------|-------|---------|------|------|--------------|--------|-------|
| Restaurant Name | Signature dish | ¥1,000 | Address | [Maps](url) | [Tabelog](url) | No | | Detailed notes... |

## 🍣 Sushi

| Name | Known For | Price | Address | Maps | Link | Reservations | Status | Notes |
|------|-----------|-------|---------|------|------|--------------|--------|-------|
...
```

Leave the **Status** column empty — the user uses this to track bookings.

---

#### Option B: CSV (`--csv`)

Write to `./japan-recs/[Area] Restaurants.csv` in the current working directory.

Columns: `Category,Name,Known For,Price,Address,Google Maps,Reservation/Review Link,Reservations,Status,Notes`

- Include the category emoji prefix in the Category column (e.g., "🍜 Ramen")
- Leave the Status column empty
- Properly escape any commas or quotes in field values

---

#### Option C: Google Sheets (`--sheet SPREADSHEET_ID`)

Use the `gws` CLI to write results to the provided spreadsheet.

1. Create a new tab named after the area (e.g., "Shinjuku Restaurants")
2. Clear any existing data if the tab already exists
3. Write data in this format:

```
Row 1: [AREA NAME] RESTAURANT RECOMMENDATIONS
Row 2: (blank)
Row 3: [Category emoji] CATEGORY NAME
Row 4: Name | Known For | Price | Address | Google Maps | Reservation/Review Link | Reservations | Status | Notes
Row 5+: Data rows
(blank row)
Row N: [Next category emoji] NEXT CATEGORY
Row N+1: Name | Known For | Price | Address | Google Maps | Reservation/Review Link | Reservations | Status | Notes
...
```

Leave the **Status** column empty.

---

### Category Emojis

Ramen 🍜, Yakiniku 🥩, Sushi 🍣, Izakaya/Bars 🍺, Tonkatsu/Soba/Teishoku 🥢, Teppanyaki/Fine Dining 🔥, Desserts & Sweets 🍡, Cafe ☕, Other 🍽️

**Always include a Desserts & Sweets section** (🍡) — research dessert spots, bakeries, mochi shops, kakigori, parfaits, taiyaki, crepes, matcha desserts, etc. specific to the area.

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
