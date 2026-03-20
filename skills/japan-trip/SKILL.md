---
name: japan-trip
description: Plan a complete Japan trip — ingest your existing picks, discover neighborhoods, research restaurants and activities, and compile everything into Google Sheets, CSV, or Markdown.
disable-model-invocation: false
user-invocable: true
allowed-tools: Bash, Agent, WebSearch, WebFetch, Write, Read
argument-hint: [city] [duration] [--from FILE|SHEET_ID|NOTION_URL] [--interests "..."] [--csv | --sheet SPREADSHEET_ID]
---

# Japan Trip Planner

Automatically plan a complete Japan trip: discover the best neighborhoods, research restaurants and activities, and compile everything into a single organized output.

## Arguments

```
/japan-trip Tokyo 7 days                          → Markdown output (default)
/japan-trip Osaka 5 days --csv                    → CSV output
/japan-trip Kyoto 3 days --sheet SHEET_ID         → Google Sheets output
/japan-trip Tokyo 10 days --interests "ramen, nightlife, vintage shopping"
/japan-trip Tokyo 7 days --from my-picks.csv      → Start from existing picks
/japan-trip Tokyo 7 days --from SHEET_ID          → Import from Google Sheet
/japan-trip Tokyo 7 days --from NOTION_URL        → Import from Notion page/database
```

- **First argument**: City or region in Japan (required)
- **Second argument**: Trip duration (required, e.g., "5 days", "1 week", "2 weeks")
- `--from SOURCE`: Import existing picks to build around (see **Importing Existing Picks** below)
- `--interests "..."`: Optional focus areas to emphasize (e.g., "fine dining, temples, anime, nightlife")
- `--csv`: Output as CSV files
- `--sheet SPREADSHEET_ID`: Output to Google Sheets (requires `gws` CLI)

If no output flag is provided, defaults to Markdown.

### Importing Existing Picks (`--from`)

The `--from` flag lets you bring your own saved restaurants, activities, or wishlist items. The skill will treat these as anchors — keeping them in the final output and filling in gaps around them with new recommendations.

Supported sources:

| Source | Format | Example |
|--------|--------|---------|
| **CSV file** | Local `.csv` or `.tsv` file | `--from my-japan-picks.csv` |
| **Excel file** | Local `.xlsx` file | `--from japan-wishlist.xlsx` |
| **Markdown file** | Local `.md` file with tables or lists | `--from saved-recs.md` |
| **Google Sheet** | Spreadsheet ID (44-char string) | `--from 1qwgFBNKK1RsYmkkt06...` |
| **Notion** | Notion page or database URL | `--from https://notion.so/...` |
| **Plain text** | Any text file with names/places | `--from notes.txt` |

The import is flexible — it will parse whatever structure it finds. At minimum it needs **place names**. Any additional info (category, area, notes, links) will be preserved and carried into the output. Columns/fields don't need to match the output schema exactly; the skill will map them intelligently.

## Workflow

### Step 0: Ingest Existing Picks (if `--from` provided)

If the user provided `--from`, read and parse their existing picks before doing anything else.

#### Reading the source

- **Local file** (`.csv`, `.tsv`, `.xlsx`, `.md`, `.txt`): Use the Read tool to read the file contents. For `.xlsx` files, use `xlsx2csv` or `python3` to convert to readable format first.
- **Google Sheet ID** (44-char alphanumeric string): Use the `gws` CLI to read all tabs:
  ```bash
  PARAMS='{"spreadsheetId":"SHEET_ID","range":"Sheet1"}'
  gws sheets spreadsheets values get --params "$PARAMS"
  ```
  Read each tab to get all data.
- **Notion URL** (contains `notion.so`): Use the Notion MCP tools if available — `notion-fetch` to read the page/database content. If Notion MCP is not available, use WebFetch to attempt to read the page.

#### Parsing picks

Parse the imported data flexibly. Look for:
- **Place names** (required) — the restaurant or activity name
- **Area/neighborhood** — where it's located (if provided)
- **Category** — type of food or activity (if provided)
- **Any notes, links, or other metadata** — preserve as-is

Create a structured list of existing picks. For each pick, note:
- Name (and Japanese name if provided)
- Area (if known — if not, look it up)
- Whether it's a restaurant or activity
- Category (if known)
- Any user notes or metadata to preserve

#### How picks influence the rest of the workflow

- **Neighborhood selection (Step 1)**: If picks are concentrated in specific areas, make sure those areas are included in the plan. The picks may also reveal areas the user cares about that wouldn't otherwise be selected.
- **Research (Step 2)**: When researching a neighborhood that has existing picks, the agent should:
  1. Research the existing picks to fill in any missing details (address, price, links, notes)
  2. Find **complementary** recommendations — different categories, price ranges, or styles that round out what the user already has
  3. Avoid recommending places that are redundant with existing picks (e.g., if the user already has 3 ramen spots in Shinjuku, prioritize other categories)
- **Output (Step 4)**: Existing picks should be marked with a `★` prefix in the Name column so the user can distinguish their original picks from new recommendations. Preserve any original notes from the user alongside new research notes.

### Step 1: Plan Neighborhoods

Spawn an Agent to determine which neighborhoods/areas to cover based on the city and trip duration.

The agent should:
1. WebSearch for "best neighborhoods to visit in [CITY] Japan", "[CITY] area guide", "where to eat and explore in [CITY]"
2. Select neighborhoods based on:
   - **Trip duration**: 1-3 days → 3-4 areas, 4-7 days → 5-7 areas, 8-14 days → 7-10 areas, 14+ days → 10-12 areas
   - **Diversity**: Mix of food-focused areas, cultural areas, shopping/entertainment areas, and local/off-the-beaten-path areas
   - **User interests**: If `--interests` was provided, weight neighborhood selection toward those interests
   - **Existing picks**: If Step 0 produced picks, ensure their neighborhoods are included. Also use the picks to infer interests (e.g., lots of ramen picks → the user loves ramen)
3. For each neighborhood, note what it's best known for (this guides the research agents)

Return a structured list like:
```
- Shinjuku: nightlife, entertainment, ramen, department stores
- Asakusa: temples, traditional culture, street food
- Shibuya: shopping, trendy restaurants, youth culture
- Tsukiji/Toyosu: seafood, market food, sushi
...
```

### Step 2: Research (Parallel)

For **each neighborhood** from Step 1, spawn an Agent (run these in parallel) to research BOTH restaurants AND activities.

If there are existing picks from Step 0 for this neighborhood, pass them to the agent so it can:
1. **Enrich** existing picks with missing details (addresses, prices, links, review quotes)
2. **Complement** them with new recommendations that fill gaps in categories, price ranges, and styles
3. **Avoid duplicates** — don't recommend places the user already picked

Each agent should search:
- Reddit (r/JapanTravel, r/Tokyo, r/JapanFood, r/japanlife)
- Google Reviews / TripAdvisor
- Tabelog (for restaurants)
- Food blogs (Ramen Adventures, Food Sake Tokyo, ByFood, etc.)
- TimeOut Tokyo, SAVOR JAPAN, Japan Guide, Japan Travel
- Social media recommendations
- Activity-specific sites (Klook, Viator, Google Things to Do)

#### Restaurant Research

For each restaurant, gather:
- **Name** (English + Japanese if available)
- **Category** (Ramen, Sushi, Yakiniku, Izakaya, Soba, Tonkatsu, Teppanyaki, Fine Dining, Desserts & Sweets, Cafe, etc.)
- **What it's known for** (signature dish or experience)
- **Price range** (Budget / Mid-range / Splurge, with yen amounts when available)
- **Address** (English, with Japanese building name if applicable)
- **Whether reservations are needed**
- **Why people recommend it** (specific praise from forums/reviews — detailed: include quotes, what makes it stand out, atmosphere, tips like best time to go or what to order)
- **Google Maps link**
- **Reservation/review link** (Tabelog, TripAdvisor, or official site)

Aim for 10-15 restaurants per neighborhood with a mix of budget, mid-range, and splurge across different food categories. **Always include desserts/sweets spots.**

#### Activity Research

For each activity/attraction, gather:
- **Name** (English + Japanese if available)
- **Category** (Temples & Shrines, Museums, Shopping, Nightlife, Parks & Nature, Markets, Entertainment, Day Trips, Experiences, etc.)
- **What it is** (brief description of the attraction/experience)
- **Duration** (how long to spend: 30 min, 1-2 hours, half day, full day)
- **Cost** (Free / ¥ amount / varies)
- **Address** (English)
- **Best time to visit** (morning, afternoon, evening, specific days, seasonal)
- **Why people recommend it** (specific praise, what makes it special, tips)
- **Google Maps link**
- **Website/booking link**

Aim for 8-12 activities per neighborhood. Include a mix of:
- Must-see landmarks and temples/shrines
- Shopping (department stores, vintage, specialty, markets)
- Nightlife and entertainment
- Parks, gardens, scenic spots
- Unique experiences (cooking classes, tea ceremonies, arcades, etc.)
- Hidden gems and local favorites

### Step 3: Address Lookup

For any restaurants or activities missing addresses from Step 2, spawn an Agent to look up the missing addresses. Include building names in Japanese when available.

### Step 4: Output Results

Output based on the user's chosen format. Organize by **neighborhood**, with restaurants and activities separated within each neighborhood section.

---

#### Option A: Markdown (default)

Write to `./japan-trip/[City] Trip Plan.md`.

Format:

```markdown
# [CITY] Trip Plan ([DURATION])

## Shinjuku

### 🍜 Restaurants

#### Ramen

| Name | Known For | Price | Address | Maps | Link | Reservations | Status | Notes |
|------|-----------|-------|---------|------|------|--------------|--------|-------|
| Restaurant Name | Signature dish | ¥1,000 | Address | [Maps](url) | [Tabelog](url) | No | | Detailed notes... |

#### Sushi
...

#### 🍡 Desserts & Sweets
...

### 🏯 Activities

#### Temples & Shrines

| Name | What It Is | Duration | Cost | Address | Best Time | Maps | Link | Status | Notes |
|------|-----------|----------|------|---------|-----------|------|------|--------|-------|
| Meiji Shrine | Major Shinto shrine in forested grounds | 1-2 hours | Free | ... | Morning | [Maps](url) | [Site](url) | | Notes... |

#### Shopping
...

---

## Shibuya
...
```

Leave **Status** columns empty — the user uses these to track visits/bookings.

---

#### Option B: CSV (`--csv`)

Write two files per area to `./japan-trip/`:
- `[City] - [Area] Restaurants.csv`
- `[City] - [Area] Activities.csv`

Restaurant columns: `Category,Name,Known For,Price,Address,Google Maps,Reservation/Review Link,Reservations,Status,Notes`

Activity columns: `Category,Name,What It Is,Duration,Cost,Address,Best Time,Google Maps,Website/Booking Link,Status,Notes`

Properly escape commas and quotes in field values. Leave Status empty.

---

#### Option C: Google Sheets (`--sheet SPREADSHEET_ID`)

Use the `gws` CLI to write results to the provided spreadsheet.

For each neighborhood, create **two tabs**:
- `[Area] Restaurants` — restaurant data
- `[Area] Activities` — activity data

Also create a **Trip Overview** tab as the first tab with:
- Row 1: `[CITY] TRIP PLAN — [DURATION]`
- Row 2: (blank)
- Row 3: Headers: `Neighborhood | Known For | # Restaurants | # Activities`
- Row 4+: Summary of each neighborhood

##### Restaurant tab format
```
Row 1: [AREA] RESTAURANTS
Row 2: (blank)
Row 3: [Category emoji] CATEGORY NAME
Row 4: Name | Known For | Price | Address | Google Maps | Link | Reservations | Status | Notes
Row 5+: Data rows
(blank row between categories)
```

##### Activity tab format
```
Row 1: [AREA] ACTIVITIES
Row 2: (blank)
Row 3: [Category emoji] CATEGORY NAME
Row 4: Name | What It Is | Duration | Cost | Address | Best Time | Google Maps | Link | Status | Notes
Row 5+: Data rows
(blank row between categories)
```

Leave **Status** columns empty.

---

### Category Emojis

**Restaurants:** Ramen 🍜, Yakiniku 🥩, Sushi 🍣, Izakaya/Bars 🍺, Tonkatsu/Soba/Teishoku 🥢, Teppanyaki/Fine Dining 🔥, Desserts & Sweets 🍡, Cafe ☕, Other 🍽️

**Activities:** Temples & Shrines 🏯, Museums 🏛️, Shopping 🛍️, Nightlife 🌙, Parks & Nature 🌸, Markets 🏪, Entertainment 🎮, Day Trips 🚃, Experiences 🎭, Other 📍

### Links

For each entry, include two links:

1. **Google Maps** — construct as: `https://www.google.com/maps/search/?api=1&query=NAME+AREA+CITY`
   - URL-encode the name (spaces become `+`)

2. **Website/Booking link** — use the best available:
   - For restaurants: Tabelog (preferred), TripAdvisor, official site, booking platform (TableCheck, OMAKASE, Klook)
   - For activities: Official website, Klook, Viator, Japan Guide page, TripAdvisor
   - Fallback: Google search URL `https://www.google.com/search?q=NAME+AREA+CITY`

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
