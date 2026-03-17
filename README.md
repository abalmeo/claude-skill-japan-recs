# japan-recs

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that researches Japan restaurant recommendations from Reddit, Tabelog, TripAdvisor & more, then outputs to Google Sheets, CSV, or Markdown.

[![Claude Code Compatible](https://img.shields.io/badge/Claude_Code-Skill-blue)](https://docs.anthropic.com/en/docs/claude-code)

## What it does

Give it an area in Japan, and it will:

1. Research restaurants across Reddit (r/JapanTravel, r/Tokyo), Tabelog, TripAdvisor, food blogs, and more
2. Gather details: name (EN + JP), category, signature dishes, price range, address, reservation info, and real reviewer quotes
3. Output 15-25 recommendations organized by category (Ramen, Sushi, Izakaya, Desserts, etc.) with Google Maps links and booking URLs

## Install

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/abalmeo/claude-skill-japan-recs.git ~/.claude/skills/japan-recs
```

Or manually copy `SKILL.md` to `~/.claude/skills/japan-recs/SKILL.md`.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- For Google Sheets output: [`gws` CLI](https://github.com/nicholasgasior/gws) installed and authenticated

## Usage

### Markdown (default — no setup required)

```
/japan-recs Shinjuku
```

Creates `./japan-recs/Shinjuku Restaurants.md` with formatted tables.

### CSV

```
/japan-recs Shinjuku --csv
```

Creates `./japan-recs/Shinjuku Restaurants.csv`.

### Google Sheets

```
/japan-recs Shinjuku --sheet 1qwgFBNKK1RsYmkkt06tzLo0B21ouD49o78Pt8YWXa0E
```

Creates a new tab "Shinjuku Restaurants" in the specified spreadsheet.

## Example Output

Each output format includes these columns per restaurant:

| Name | Known For | Price | Address | Maps | Link | Reservations | Status | Notes |
|------|-----------|-------|---------|------|------|--------------|--------|-------|
| Fuunji (風雲児) | Tsukemen dipping ramen | ¥1,000 | 3-16-4 Shinjuku | [Maps](https://www.google.com/maps/search/?api=1&query=Fuunji+Shinjuku+Tokyo) | [Tabelog](https://tabelog.com/tokyo/A1304/A130401/13001257/) | No | | "Best tsukemen in Shinjuku. Rich pork/fish broth. Expect 30min wait at lunch." |

Restaurants are grouped by category with emojis:
🍜 Ramen · 🍣 Sushi · 🥩 Yakiniku · 🍺 Izakaya · 🥢 Tonkatsu/Soba · 🔥 Fine Dining · 🍡 Desserts & Sweets · ☕ Cafe

## License

[MIT](LICENSE)
