---
name: travel-map
description: Generate a color-coded KML map from a Google Sheet with travel data (itinerary, restaurants, bars, shopping). Produces a file importable into Google My Maps.
disable-model-invocation: false
user-invocable: true
allowed-tools: Bash, Agent
argument-hint: [Google Sheet URL, spreadsheet ID, or sheet name to search for]
---

# Travel Map — KML Generator from Google Sheets

Read a travel planning Google Sheet and generate a **color-coded KML file** that can be imported into Google My Maps.

## Style & Color-Coding Reference

Google My Maps **ignores standard KML `<Style>` blocks** for color-coding in most real-world imports (large files, nested folders, address-based placemarks). After extensive testing, the reliable approach uses **two complementary methods**:

### Method 1: ExtendedData "category" field (primary — enables "Style by" in Google My Maps)

Add an `<ExtendedData>` element to each Placemark with a `category` field:

```xml
<ExtendedData>
  <Data name="category">
    <value>Restaurant</value>
  </Data>
</ExtendedData>
```

After importing, the user clicks the layer's style dropdown → **"Style by" → "category"** to get 6 groups, then assigns colors per group.

### Method 2: Name prefix (secondary — visual fallback)

Prefix each placemark name with the category: `[Restaurant] Fuunji`, `[Reservation] ★ Muscle Girls [BOOKED]`, `[Bar] Bar Rpm`, etc. This makes categories visible even without color-coding.

### Method 3: KML Style + StyleMap (included but unreliable)

Include Google My Maps proprietary styles as a best-effort fallback. These work in small files but not reliably in large imports:

- Icon URL: `https://www.gstatic.com/mapspro/images/stock/503-wht-blank_maps.png`
- Style ID convention: `icon-1899-{HEX}-normal` / `icon-1899-{HEX}-highlight`
- StyleMap: pairs normal + highlight; Placemarks reference via `<styleUrl>#icon-1899-{HEX}</styleUrl>`
- Color format: `ff` + RRGGBB (NOT standard KML AABBGGRR)
- hotSpot: `<hotSpot x="32" xunits="pixels" y="64" yunits="insetPixels" />`

### Color Mapping

| Category | Prefix | Color | Hex |
|---|---|---|---|
| reservation | `[Reservation]` | Red | DB4437 |
| planned | `[Planned]` | Orange | FF8B00 |
| restaurant | `[Restaurant]` | Yellow | F4EB37 |
| bar | `[Bar]` | Purple | 7B1FA2 |
| shopping | `[Shopping]` | Blue | 0288D1 |
| accommodation | `[Accommodation]` | Green | 0F9D58 |

## Workflow

### Step 1: Resolve Spreadsheet

Parse the user's argument to get a spreadsheet ID:

- **URL**: Extract ID via regex from `https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/...`
- **ID**: If it's a long alphanumeric string (with hyphens/underscores), use directly
- **Name search**: Search Google Drive:
  ```bash
  gws drive files list --params '{"q": "name contains '\''ARG'\'' and mimeType = '\''application/vnd.google-apps.spreadsheet'\''", "fields": "files(id, name)"}'
  ```

Validate access and get metadata (title + tab names):
```bash
PARAMS='{"spreadsheetId":"SHEET_ID","fields":"properties.title,sheets.properties"}'
gws sheets spreadsheets get --params "$PARAMS"
```

### Step 2: Classify Tabs

Auto-classify each tab by keyword matching on the tab name (case-insensitive):

| Keywords in tab name | Category | Pin Color |
|---|---|---|
| restaurant, food, dining, eat | `restaurant` | Yellow |
| bar, drink, nightlife, pub, hop | `bar` | Purple |
| shop, shopping, store | `shopping` | Blue |
| itinerary, schedule, plan, activity | `planned` | Orange |
| accommodation, hotel, airbnb, stay | `accommodation` | Green |
| flight, train, transport, expense, budget, cost, question, reference, old, checklist | `skip` | — |

Rules:
- Hidden tabs (`hidden: true` in sheet properties) → skip by default
- Unmatched tabs → ask the user to classify or skip
- **Present the full classification to the user for confirmation before proceeding**

### Step 3: Read & Parse Data

For each included tab, read all values:
```bash
PARAMS='{"spreadsheetId":"SHEET_ID","range":"TAB_NAME"}'
gws sheets spreadsheets values get --params "$PARAMS"
```

**Column detection** — Find the header row (first row with 3+ non-empty cells), then match columns case-insensitively:

| Data field | Column name matches |
|---|---|
| Name | "name", "activity", "place" — or fall back to first text column |
| Address | "address", "location" |
| Google Maps URL | "google maps", "maps", "map link", "google maps link" |
| Status | "status", "booking" |
| Notes | "notes", "known for", "description" |
| Price | "price", "cost" |
| Time | "time" |
| Area | "area", "neighborhood" |

**Itinerary tab special handling** (tabs classified as `planned`):
- Detect day headers: rows matching day-of-week patterns like "FRIDAY", "SATURDAY", "DAY 1", etc.
- Only create placemarks from rows that have BOTH a name/activity AND an address
- Use the most recent day header as context in the placemark description
- Skip section headers, audible/backup lists, and reference-only rows

**Row skip logic** (apply to ALL tabs):
- Skip rows where name AND address are both empty
- Skip rows that look like category headers (emoji + uppercase text with no address)
- Skip rows that are just notes/lists without a place name

### Step 4: Generate KML via Python

Write parsed data to a temp JSON file, then run an inline Python script.

The Python script MUST:

1. **Use `xml.etree.ElementTree`** for all XML generation — NO string concatenation for XML content. This prevents escaping bugs with `&`, `<`, `>` in names/descriptions.

2. **Color-coding** — apply all three methods from the Style Reference:
   - **Name prefix**: Prefix each placemark name with `[Category]` (e.g., `[Restaurant] Fuunji`)
   - **ExtendedData**: Add `<ExtendedData><Data name="category"><value>Category</value></Data></ExtendedData>` to each placemark
   - **KML styles**: Generate `Style` + `StyleMap` blocks at Document level using gstatic icon + color tint (best-effort; see Style Reference for details)

3. **Reservation override**: Any row where the Status column matches (case-insensitive) "booked", "reserved", or "confirmed" gets `#reservation` style — regardless of which tab/category it came from. This is CRITICAL.

4. **Coordinate extraction** from Google Maps URLs:
   - Try regex `@(-?\d+\.\d+),(-?\d+\.\d+)` to extract lat/lng from the URL
   - If coordinates found → use `<Point><coordinates>lng,lat,0</coordinates></Point>`
   - If no coordinates but address exists → use `<address>` element
   - If neither (only a search query URL) → **skip the placemark** and warn (Google My Maps cannot geocode query-only URLs)

5. **Folder organization**:
   - One KML `<Folder>` per tab
   - For restaurant tabs: if category header rows exist (emoji + uppercase text like "🍜 RAMEN"), create sub-`<Folder>` elements for each category group

6. **Description assembly**: Build from available columns:
   ```
   Time | Price | Area
   Notes/Known For
   Status: {status}
   Maps: {google_maps_url}
   ```
   Only include fields that have values. Use `<description>` with CDATA or properly escaped text.

7. **Output**: Write to `~/Desktop/{sanitized_title}.kml` where sanitized_title strips unsafe filesystem characters.

### Step 5: Report Results

After generating the KML, print:
- Output file path
- Placemark counts by category/color (table format)
- Any rows skipped and why (missing address, vague location, etc.)
- Warnings for vague addresses matching patterns like "Near X", single ward name, just "Tokyo, Japan"
- Import instructions:

```
To import: Google My Maps → Create new map → Import → upload the KML file
```

## GWS CLI Reference

### Shell escaping
Always pass params via a variable to avoid `!` escaping issues:
```bash
PARAMS='{"spreadsheetId":"SHEET_ID","range":"TabName!A1"}'
gws sheets spreadsheets values get --params "$PARAMS"
```

### Get spreadsheet metadata
```bash
PARAMS='{"spreadsheetId":"SHEET_ID","fields":"properties.title,sheets.properties"}'
gws sheets spreadsheets get --params "$PARAMS"
```

### Read tab data
```bash
PARAMS='{"spreadsheetId":"SHEET_ID","range":"Tab Name"}'
gws sheets spreadsheets values get --params "$PARAMS"
```

### Read multiple ranges at once
```bash
PARAMS='{"spreadsheetId":"SHEET_ID","ranges":["Tab1","Tab2","Tab3"]}'
gws sheets spreadsheets values batchGet --params "$PARAMS"
```

### Search Google Drive for spreadsheets
```bash
gws drive files list --params '{"q": "name contains '\''SEARCH_TERM'\'' and mimeType = '\''application/vnd.google-apps.spreadsheet'\''", "fields": "files(id, name)"}'
```
