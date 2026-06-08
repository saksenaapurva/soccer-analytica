---
name: soccer-analytica-audit
description: >
  Monthly Instagram Reels audit skill for the @soccer_analytica channel.
  Use this skill whenever the task is to update, refresh, or audit the Soccer Analytica dashboard —
  including scraping Instagram Insights for each reel, updating the SCRAPED data array,
  syncing data to Supabase, updating the Cowork artifact (id: soccer-analytica), and pushing to GitHub.
  Triggers on: "run monthly audit", "update soccer analytica dashboard", "scrape Instagram insights",
  "refresh the dashboard", or any scheduled task mentioning soccer_analytica.
  This skill is the single source of truth for how to reproduce the full data pipeline.
---

# Soccer Analytica — Monthly Audit Skill

This skill documents the complete workflow for scraping @soccer_analytica's Instagram Insights,
updating the dashboard HTML, syncing to Supabase, and pushing to GitHub. Follow every step in order.

---

## Account Details

- **Instagram handle**: @soccer_analytica
- **Platform**: Instagram Reels only
- **Cowork artifact ID**: `soccer-analytica`
- **Dashboard HTML file**: `soccer-analytica.html` in the outputs folder
  - Known path: `/Users/apurvasaksena/Library/Application Support/Claude/local-agent-mode-sessions/846df5fb-81ed-4157-b444-93123bf4b68f/210a8a0b-70cd-4870-8c05-1f2b8d106bc3/local_f8b5c574-da74-4202-b98d-9639ea7196f7/outputs/soccer-analytica.html`
  - If that path doesn't exist, search for `soccer-analytica.html` in the user's home directory
- **GitHub username**: `saksenaapurva`
- **GitHub repo**: `soccer-analytica` (private repo, branch: `main`)
- **Supabase project ID**: `byxiutawntkirnwtbpju`
- **Supabase table**: `reels`
- **Supabase org**: `clinical-notes` (org ID: `reiygesdkdcmkzvuklqa`)

---

## Prerequisites

Before starting, verify:
1. Chrome is open and the user is logged into Instagram at instagram.com
2. The `mcp__Claude_in_Chrome__*` tools are available (Chrome MCP)
3. The dashboard HTML file exists at the path above

If Chrome is not open or Instagram is not logged in, stop and tell the user — you cannot proceed without a live authenticated session.

---

## Step 1 — Read the Current SCRAPED Array

Read the dashboard HTML file and extract the existing `SCRAPED` array. This tells you:
- Which reels are already tracked (by their `url` shortcode field)
- What data was recorded in the last run
- The current reel count

Look for the JavaScript block starting with `const SCRAPED = [` and parse out all existing entries.

---

## Step 2 — Navigate to the Instagram Profile

Use Chrome MCP to navigate to the profile:
```
navigate to: https://www.instagram.com/soccer_analytica/
```

Wait for the page to load (use `wait` or check `get_page_text` for profile content).

---

## Step 3 — Get All Reel Shortcodes

From the profile page, collect the shortcodes of all visible reels. Each reel URL looks like:
`https://www.instagram.com/reel/{SHORTCODE}/`

Use JavaScript to extract all reel links currently visible:
```javascript
[...document.querySelectorAll('a[href*="/reel/"]')]
  .map(a => a.href.match(/\/reel\/([^\/]+)/)?.[1])
  .filter(Boolean)
  .filter((v, i, arr) => arr.indexOf(v) === i)
```

Compare against the existing `SCRAPED` array. Any shortcode not already in SCRAPED is a **new reel** — add it to the end of the list to scrape. All existing shortcodes should also be re-scraped to get fresh data.

Note: Instagram may only show 12 reels at first. Scroll down to load more before extracting. Keep scrolling until no new reels appear.

---

## Step 4 — Scrape Insights for Each Reel

For each reel shortcode, follow this exact sequence:

### 4a. Navigate to the reel
```
navigate to: https://www.instagram.com/p/{SHORTCODE}/
```
Wait 2 seconds for the page to fully render.

**IMPORTANT**: Use `/p/{SHORTCODE}/` — NOT `/reel/{SHORTCODE}/`. The "View insights" button only appears on the `/p/` URL format.

### 4b. Click "View insights"
Use JavaScript — do NOT try to click by coordinate or ref, as the button is a `div[role="button"]`:
```javascript
const el = [...document.querySelectorAll('[role="button"]')]
  .find(e => e.textContent?.trim() === 'View insights');
el?.click();
```

If `el` is null, the user may not be logged in or the reel belongs to a different account. Skip with a note.

### 4c. Wait and extract
Wait 2–3 seconds for the insights panel to load, then call `get_page_text` to extract the content.

### 4d. Parse the metrics
The insights page text contains metrics in this approximate order. Parse them carefully — the exact layout can vary:

```
Views
Views
{views_count}

Accounts reached
{reached_count}

Accounts engaged
{engaged_count}

Likes
{likes_count}

Comments
{comments_count}

Saves
{saves_count}

Shares
{shares_count}

Follows
{follows_count}

Profile visits
{profile_visits_count}

Plays
{plays_count}

Reach
Followers  {follower_pct}%
Non-followers  {non_follower_pct}%
```

**Key parsing rules:**
- Views appear as: `Views\nViews\n{number}` — take the number after the second "Views"
- If views show as `--` or `0` and the reel is very old (2020–2021 era), set `noViewData: true`
- `followerPct` = the **Followers** percentage (not non-followers). e.g. if it says "Followers 0.1%", set `followerPct: 0.1`
- All counts are integers. If a metric shows `--` or is absent, use `0`

### 4e. Build the data object
```javascript
{
  id: {sequential_id},          // keep existing id if re-scraping, assign next integer for new reels
  url: "{SHORTCODE}",
  title: "{reel_title_from_caption}",  // first ~60 chars of caption, or keep existing title
  views: {number},
  reached: {number},
  likes: {number},
  comments: {number},
  saves: {number},
  shares: {number},
  engaged: {number},
  followerPct: {number_or_null},  // null if not shown
  category: "{keep_existing_or_infer}",
  noViewData: true,  // only include if views data is unavailable
  hook: "{keep_existing_analysis}",
  tactics: "{keep_existing_analysis}",
  growth: "{keep_existing_analysis}"
}
```

**Important**: Preserve existing `hook`, `tactics`, `growth`, `category`, and `title` text for reels that are already in SCRAPED. Only update the numeric metrics. For new reels, write fresh analysis in those fields (see Step 6).

---

## Step 5 — Update the Dashboard HTML

After scraping all reels:

### 5a. Update the SCRAPED array
Replace the entire `const SCRAPED = [...]` block in the HTML file with the freshly scraped data. Use the `Edit` tool with a careful replacement.

### 5b. Update the summary stats
Recalculate and update the stats row cards:
- **Total Reels**: count of all entries
- **Total Views**: sum of all views (format as K if ≥1000)
- **Median Views**: median of all non-zero view counts
- **Viral Hit**: the single highest-view reel + its views count
- **Total Reach**: sum of all `reached` values

Recalculate the alert banner numbers:
- Total likes / shares / saves across all reels
- Median excluding the viral outlier
- Most recent 3 posts and whether they're under 50 views

### 5c. Update the Pro Analyst Scorecard
Recalculate:
- **Interaction Rate**: `(total_likes + total_comments + total_shares + total_saves) / total_reach × 100`
- **Save Rate**: `total_saves / total_reach × 100`
- The 4 algorithm signal cards (exact counts)
- The audit checklist (re-evaluate each item based on current data)

---

## Step 6 — Write Analysis for New Reels

For any **new reels** not previously in SCRAPED, write analysis in three fields:

**`hook`** — Analyse the opening 3 seconds and title as a viral short-form specialist would:
- Does the title create a curiosity gap?
- Is there a question or provocative claim?
- What would make it better?

**`tactics`** — Analyse as a football content authority:
- What content pillar does this serve?
- How does it fit the channel's analytical brand?
- What's the tactical content angle?

**`growth`** — Analyse as an algorithm/distribution expert:
- Why did this get the views it got?
- What should the next steps be?
- What timing/format/hashtag improvements could help?

Keep each analysis to 2–4 sentences. Base it on the actual metrics scraped — reference the real numbers.

---

## Step 7 — Sync to Supabase

After the dashboard HTML is updated, upsert all scraped reels into the Supabase `reels` table.

**Project ID**: `byxiutawntkirnwtbpju`  
**Table**: `reels`

Use `mcp__supabase__execute_sql` with an UPSERT. Build one query covering all reels:

```sql
INSERT INTO reels (id, url, title, views, reached, likes, comments, saves, shares, engaged,
                   follower_pct, category, no_view_data, hook, tactics, growth, updated_at)
VALUES
  ({id}, '{url}', '{title}', {views}, {reached}, {likes}, {comments}, {saves}, {shares},
   {engaged}, {follower_pct_or_NULL}, '{category}', {true|false}, '{hook}', '{tactics}', '{growth}', now()),
  -- repeat for each reel
ON CONFLICT (id) DO UPDATE SET
  views        = EXCLUDED.views,
  reached      = EXCLUDED.reached,
  likes        = EXCLUDED.likes,
  comments     = EXCLUDED.comments,
  saves        = EXCLUDED.saves,
  shares       = EXCLUDED.shares,
  engaged      = EXCLUDED.engaged,
  follower_pct = EXCLUDED.follower_pct,
  updated_at   = now();
  -- NOTE: hook/tactics/growth are NOT updated on conflict — preserve existing analysis
```

**SQL escaping rules:**
- Escape single quotes in all text fields by doubling them: `it's` → `it''s`
- Use `NULL` (unquoted) for null follower_pct values
- Use `true`/`false` (unquoted) for no_view_data boolean

**After upserting, verify with:**
```sql
SELECT COUNT(*) as total_reels,
       SUM(views) as total_views,
       SUM(reached) as total_reach,
       ROUND(SUM(likes + comments + shares + saves)::numeric / NULLIF(SUM(reached), 0) * 100, 2) as ir_pct,
       ROUND(SUM(saves)::numeric / NULLIF(SUM(reached), 0) * 100, 2) as save_rate_pct
FROM reels;
```

---

## Step 8 — Update the Cowork Artifact

After saving the HTML file, update the Cowork artifact:
```
mcp__cowork__update_artifact:
  id: "soccer-analytica"
  html_path: {path_to_soccer-analytica.html}
  update_summary: "Monthly audit: refreshed data for {reel_count} reels, {new_reels} new. Total reach: {total_reach}. IR: {interaction_rate}%."
```

---

## Step 9 — Push to GitHub

Push the updated `soccer-analytica.html` to the `soccer-analytica` GitHub repo as `index.html`:

```
mcp__github__create_or_update_file:
  owner: saksenaapurva
  repo: "soccer-analytica"
  path: "index.html"
  message: "Monthly audit — {current_date}: {reel_count} reels, {new_reels} new"
  content: {base64_encoded_html_content}
  sha: {current_file_sha}
```

To get the current SHA before overwriting:
```
mcp__github__get_file_contents:
  owner: saksenaapurva
  repo: "soccer-analytica"
  path: "index.html"
```
Extract the `sha` field from the response, then pass it to `create_or_update_file`.

---

## Step 10 — Summary Report

After completing everything, output a brief summary:

```
✅ Monthly audit complete — {date}

📊 Reels scraped: {total} ({new_count} new)
👁️  Total views: {total_views}
📡 Total reach: {total_reach}
📈 Interaction Rate: {ir}% (benchmark: 3.5%)
🔖 Save Rate: {save_rate}% (target: >2%)

New reels found: {list of new shortcodes + titles}
Database:  Supabase reels table synced ✅
Dashboard: Cowork artifact updated ✅
GitHub:    https://saksenaapurva.github.io/soccer-analytica/ updated ✅
```

---

## Error Handling

| Problem | Action |
|---------|--------|
| "View insights" button not found | User may not be logged in. Stop and report. |
| Reel shows `--` for views | Set `noViewData: true`, set views/reached to 0, keep other metrics |
| get_page_text returns empty | Wait 3s and retry once. If still empty, skip reel and note it. |
| GitHub SHA mismatch | Re-fetch the file's SHA and retry |
| Supabase upsert fails | Check for unescaped single quotes — double all apostrophes in text fields |
| New reel count > 5 | Still process all — just takes longer |

---

## Data Schema Reference

```javascript
// Full SCRAPED entry schema (dashboard JS)
{
  id: Number,           // sequential, 1-based, don't change for existing reels
  url: String,          // Instagram shortcode (e.g., "DXuxvGHEraO")
  title: String,        // reel caption / topic description
  views: Number,        // play count from insights
  reached: Number,      // accounts reached
  likes: Number,
  comments: Number,
  saves: Number,
  shares: Number,
  engaged: Number,      // accounts engaged
  followerPct: Number|null,  // % of reach that are followers (not non-followers)
  category: String,     // content category (e.g., "UCL / Trending", "Chelsea History")
  noViewData: Boolean,  // true only for pre-2022 reels with no view data
  hook: String,         // Hook Master analysis
  tactics: String,      // Tactics Expert analysis
  growth: String        // Growth Analyst analysis
}
```

```sql
-- Supabase reels table columns
id               integer PRIMARY KEY
url              text             -- Instagram shortcode
title            text
views            integer
reached          integer
likes            integer
comments         integer
saves            integer
shares           integer
engaged          integer
follower_pct     numeric(5,2)     -- nullable
category         text
no_view_data     boolean
hook             text
tactics          text
growth           text
interaction_rate numeric(6,4)     -- auto-calculated: (likes+comments+shares+saves)/reached*100
save_rate        numeric(6,4)     -- auto-calculated: saves/reached*100
scraped_at       timestamptz      -- first insert time
updated_at       timestamptz      -- updated each audit run
```

---

## Niche Context (for writing new reel analysis)

@soccer_analytica is a **football animation / mystery explainer** channel:
- Visual style: cartoon animated players, bold italic yellow-outlined text (YouTube thumbnail style)
- Best content formula: [Surprising football event] + [data that explains why] + [curiosity hook title]
- Top content pillars: UCL/trending moments, historical drama, cross-club EPL coverage, tactical analysis
- Audience rewards: questions not statements, "why did this happen?" format, animated visual style
- Algorithm weaknesses to address: saves (low rate, need 2%+), comments (near zero), posting consistency
- Benchmarks: IR target 3.5%, save rate target 2%, nano-creator engagement should be 5-10%
- Tagline: "The Data Behind The Drama"
