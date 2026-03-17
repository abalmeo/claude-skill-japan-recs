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
- For Google Sheets output: `gws` CLI (see [Google Sheets Setup](#google-sheets-setup) below)

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

## Google Sheets Setup

The `--sheet` output mode requires the `gws` CLI (`@googleworkspace/cli`), a command-line tool for interacting with Google Workspace APIs. Markdown and CSV modes work without any extra setup.

### 1. Install gws

```bash
npm install -g @googleworkspace/cli
```

Verify it installed:

```bash
gws --version
```

### 2. Create a Google Cloud project

1. Go to the [Google Cloud Console](https://console.cloud.google.com/)
2. Click the project dropdown at the top and select **New Project**
3. Give it a name (e.g., "gws-cli") and click **Create**
4. Make sure the new project is selected in the project dropdown

### 3. Enable required APIs

In your GCP project, enable these APIs under **APIs & Services > Library**:

- **Google Sheets API** — required for reading/writing spreadsheet data
- **Google Drive API** — required for creating tabs and managing spreadsheet structure

Search for each one and click **Enable**.

### 4. Configure the OAuth consent screen

Before creating credentials, you need to set up the consent screen:

1. Go to **APIs & Services > OAuth consent screen**
2. Select **External** user type (unless you have a Google Workspace org, then use Internal)
3. Fill in the required fields:
   - **App name**: anything (e.g., "gws-cli")
   - **User support email**: your email
   - **Developer contact email**: your email
4. Click **Save and Continue**
5. On the **Scopes** screen, click **Add or Remove Scopes** and add:
   - `https://www.googleapis.com/auth/spreadsheets`
   - `https://www.googleapis.com/auth/drive`
6. Click **Save and Continue**
7. On the **Test users** screen, click **Add Users** and add your Google account email
8. Click **Save and Continue**

> **Note**: While your app is in "Testing" mode, only the test users you add can authenticate. This is fine for personal use. If you skip adding yourself as a test user, the OAuth login will fail with a 403 error.

### 5. Create OAuth credentials

1. Go to **APIs & Services > Credentials**
2. Click **Create Credentials > OAuth client ID**
3. Select **Desktop app** as the application type
4. Give it a name (e.g., "gws-cli")
5. Click **Create**
6. Click **Download JSON** on the confirmation dialog
7. Save the downloaded file to `~/.config/gws/client_secret.json`:

```bash
mkdir -p ~/.config/gws
mv ~/Downloads/client_secret_*.json ~/.config/gws/client_secret.json
```

### 6. Authenticate

```bash
gws auth login
```

This opens your browser. Sign in with the Google account you added as a test user, and grant the requested permissions. You may see a "Google hasn't verified this app" warning — click **Advanced > Go to [app name]** to proceed.

### 7. Verify it's working

```bash
gws auth status
```

You should see output like:

```json
{
  "has_refresh_token": true,
  "encryption_valid": true,
  "token_valid": true,
  "scopes": [
    "https://www.googleapis.com/auth/drive",
    "https://www.googleapis.com/auth/spreadsheets",
    ...
  ]
}
```

The key things to confirm:
- `"has_refresh_token": true`
- `"token_valid": true`
- Scopes include `spreadsheets` and `drive`

### Alternative: Automated setup (requires gcloud CLI)

If you have the [gcloud CLI](https://cloud.google.com/sdk/docs/install) installed, `gws` can automate the project and credential setup:

```bash
# Create a new project and log in
gws auth setup --login

# Or use an existing project
gws auth setup --project YOUR_GCP_PROJECT_ID --login
```

### Finding your Spreadsheet ID

The spreadsheet ID is the long string in the Google Sheets URL:

```
https://docs.google.com/spreadsheets/d/1qwgFBNKK1RsYmkkt06tzLo0B21ouD49o78Pt8YWXa0E/edit
                                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                       This is the spreadsheet ID
```

Pass it with the `--sheet` flag:

```
/japan-recs Shinjuku --sheet 1qwgFBNKK1RsYmkkt06tzLo0B21ouD49o78Pt8YWXa0E
```

### Troubleshooting

| Problem | Solution |
|---------|----------|
| `403: access_denied` during login | You didn't add yourself as a test user in the OAuth consent screen (Step 4.7) |
| `403: insufficient_scope` when writing | Re-run `gws auth login` — make sure both Sheets and Drive scopes are granted |
| `API has not been enabled` error | Go back to Step 3 and enable both the Google Sheets API and Google Drive API |
| `client_secret.json not found` | Make sure the downloaded JSON is saved to `~/.config/gws/client_secret.json` |

## Example Output

Each output format includes these columns per restaurant:

| Name | Known For | Price | Address | Maps | Link | Reservations | Status | Notes |
|------|-----------|-------|---------|------|------|--------------|--------|-------|
| Fuunji (風雲児) | Tsukemen dipping ramen | ¥1,000 | 3-16-4 Shinjuku | [Maps](https://www.google.com/maps/search/?api=1&query=Fuunji+Shinjuku+Tokyo) | [Tabelog](https://tabelog.com/tokyo/A1304/A130401/13001257/) | No | | "Best tsukemen in Shinjuku. Rich pork/fish broth. Expect 30min wait at lunch." |

Restaurants are grouped by category with emojis:
🍜 Ramen · 🍣 Sushi · 🥩 Yakiniku · 🍺 Izakaya · 🥢 Tonkatsu/Soba · 🔥 Fine Dining · 🍡 Desserts & Sweets · ☕ Cafe

## License

[MIT](LICENSE)
