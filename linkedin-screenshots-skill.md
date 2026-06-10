---
name: linkedin-screenshots
description: Captures screenshots of LinkedIn posts for a given month, pulls analytics data, and builds/updates the monthly SMM HTML report. Use when the user wants to screenshot posts, pull LinkedIn analytics, or build/update a monthly SMM report.
allowed-tools: Bash(playwright-cli:*), Bash(grep:*), Bash(mkdir:*), Bash(mv:*), Bash(python3:*), Bash(ls:*), Bash(cp:*), Read, Edit
---

# LinkedIn Screenshots & SMM Report Builder

Captures posts, pulls analytics data, and builds the monthly SMM HTML report for Devoted Studios (STUDIO) and Fusion by Devoted (PLATFORM).

---

## ⚠️ Brand name conventions (ALWAYS apply)

| Real name | Use in report |
|-----------|--------------|
| Devoted Studios | **STUDIO** |
| Fusion by Devoted / Devoted Fusion | **PLATFORM** |
| Ninel Anderson | **Ninel** (drop Anderson) |
| Password gate logo | **SMM REPORT** (no brand prefix) |

---

## Report file naming & structure

| File | Period | Base from |
|------|--------|-----------|
| `social-media-report-2026_6-FINAL.html` | April 2026 (full month) | — |
| `social-media-report-2026_7-FINAL.html` | May 1–15 (biweekly) | Copy of _6 |
| `social-media-report-2026_8-FINAL.html` | May 2026 (full month) | Copy of _6 |
| `social-media-report-2026_N-FINAL.html` | Next month | Copy of previous FINAL |

**Always copy the previous FINAL as base, then update data. Never build from scratch.**

```bash
cp "social-media-report-2026_8-FINAL.html" "social-media-report-2026_9-FINAL.html"
```

---

## Tab 1 structure (FINAL format — approved June 2026)

```
Layer 1 · Scorecard
  ┌──────────────────────┬──────────────────────┐
  │  STUDIO (dark)       │  PLATFORM (cyan)     │
  │  LinkedIn company    │  LinkedIn company    │
  │  Paid boost          │  Paid boost          │
  │  Business outcomes   │  Business outcomes   │
  └──────────────────────┴──────────────────────┘
  ┌─────────────────────────────────────────────┐
  │  Ninel · LinkedIn (purple header)  full-width│
  │  KPIs + Top posts + Takeaways               │
  └─────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────┐
  │  STUDIO · YouTube (red header)  full-width  │
  └─────────────────────────────────────────────┘

Layer 2 · Diagnosis  (driver grid 2×2 + verdict)
Layer 3 · Decisions  (priority cards)
Strategic Backlog    (Ninel's feedback)
```

Tabs 2–6 = Layer 4 detail data (untouched).

---

## Channel URLs & admin pages

| Channel | Public URL | Admin URL |
|---------|-----------|-----------|
| STUDIO LinkedIn | `https://ua.linkedin.com/company/devotedstudios` | `https://www.linkedin.com/company/18860165/admin/` |
| PLATFORM LinkedIn | `https://ua.linkedin.com/company/devotedfusion` | `https://www.linkedin.com/company/102351309/admin/` |
| Ninel LinkedIn | `https://ca.linkedin.com/in/ninel-anderson-92674455/` | — personal |
| YouTube | `https://www.youtube.com/@Devoted-Talks` | YouTube Studio |
| STUDIO GA4 | — | Account: Devoted Studios |
| PLATFORM GA4 | — | Account: Devoted CG |

---

## Step 1 — Screenshot posts (monthly)

### Use persistent profile (always — gives admin access)

```bash
cd "/Users/juls/Documents/playwright-cli-main/SMM REPORT"
playwright-cli open "https://www.linkedin.com/company/18860165/admin/page-posts/published/" --headed --persistent
```

### Identify May posts via URN timestamp mapping

LinkedIn uses relative timestamps. Map from today's date:
- `1d–6d` → this week (June)
- `1w` → ~June 1
- `2w` → ~May 25
- `3w` → ~May 18
- `4w` → ~May 11
- `1mo` → ~May 8

**Get all URNs from DOM:**
```bash
playwright-cli eval "
Array.from(document.querySelectorAll('[data-urn]')).map(function(el){
  var t = el.querySelector('time');
  return el.getAttribute('data-urn') + '||' + (t ? t.getAttribute('datetime') || t.textContent.trim() : '');
}).join('\n')
" 2>&1 | grep "urn:li:activity"
```

**Identify month boundary using URN IDs:**
- URN IDs are sequential timestamps — higher = newer
- Find last URN from previous month's JSON (e.g. April last = 7455316872310243329)
- Find June boundary (~6d from today)
- May posts = URNs between those two values

### Screenshot each post by URL

```bash
# Navigate to each post's admin analytics URL to confirm content, then screenshot
playwright-cli goto "https://www.linkedin.com/feed/update/urn:li:activity:<URN>/"
sleep 3
playwright-cli snapshot > /dev/null 2>&1
SNAP=$(ls -t .playwright-cli/page-*.yml | head -1)

# Find the Feed post listitem ref
REF=$(grep -B2 "Feed post" "$SNAP" | grep "listitem \[ref=" | tail -1 | grep -o "ref=e[0-9]*" | sed 's/ref=//')

playwright-cli screenshot "$REF" --filename="Linkedin Devoted Studios May/post-01-may-topic.png"
```

**Filename convention:** `post-<NN>-<mon>-<short-topic>.png`

### Folder naming

```
Linkedin Devoted Studios May/
Linkedin Devoted Studios April/
Linkedin Ninel May/
Linkedin Devoted Fusion/
Linkedin DS May 1-15/
```

---

## Step 2 — Pull LinkedIn analytics

### STUDIO & PLATFORM (company pages via admin)

Navigate to admin analytics and extract data:

```bash
# Followers
playwright-cli goto "https://www.linkedin.com/company/18860165/admin/analytics/followers/"
sleep 4
playwright-cli snapshot
SNAP=$(ls -t .playwright-cli/page-*.yml | head -1)
grep -n "follower\|Follower\|Total\|New followers" "$SNAP" | grep -iv "notif\|button\|nav" | head -15
```

**For post metrics — use individual post admin analytics:**
```bash
playwright-cli goto "https://www.linkedin.com/company/18860165/admin/post-analytics/urn:li:activity:<URN>"
sleep 3
playwright-cli snapshot
```

Extract from snapshot:
```python
# Pattern: paragraph VALUE then paragraph LABEL
# e.g.: paragraph "789" then paragraph "Impressions"
```

**For the full content analytics table:**
- Navigate to: `https://www.linkedin.com/company/18860165/admin/analytics/updates/`
- Set date range filter to target month
- Scroll table container: `playwright-cli eval "document.querySelector('table').parentElement.scrollTop = 5000"`
- LinkedIn table uses virtual scroll — only ~10 rows render at once

**Export button** (preferred for complete data):
```bash
SNAP=$(ls -t .playwright-cli/page-*.yml | head -1)
grep -n "Export" "$SNAP"
playwright-cli click <export-ref>
# File downloads to ~/Downloads/
```

### Ninel (personal profile — Excel export required)

LinkedIn personal profiles don't expose analytics via DOM. Always request Excel:

> "Download LinkedIn Content Analytics for Ninel's profile → Creator Mode → Analytics → Content → Export → May 1–31"

File name: `AggregateAnalytics_Ninel Anderson_2026-05-01_2026-05-31.xlsx`

**Parse with pandas:**
```python
import pandas as pd
xl = pd.read_excel("AggregateAnalytics_Ninel Anderson_...xlsx", sheet_name=None)

# DISCOVERY: total impressions + members reached
disc = xl['DISCOVERY']

# FOLLOWERS: daily new followers + end total
fol = xl['FOLLOWERS']  # row 0 = total; rows 2+ = daily

# TOP POSTS: cols 0-2 = by engagement, cols 4-6 = by impressions
tp = xl['TOP POSTS']

# DEMOGRAPHICS: company, location, seniority, industry
demo = xl['DEMOGRAPHICS']
```

**Key Ninel audience facts (May 2026):**
- 37% Computer Games industry
- 40% Senior / Director / CXO seniority
- Top companies: Riot Games, Epic, Blizzard, EA, Scopely, Microsoft, Meta, Sony IE
- Top locations: LA (15%), SF Bay (6%), Seattle (4%)

---

## Step 3 — LinkedIn Ads CSV

Two accounts have ad data:

| Account | File pattern | Covers |
|---------|-------------|--------|
| DevotedStudios | `devotedstudios 1.csv` + `devoted studios 2.csv` | STUDIO + Ninel + BD personal accounts |
| Devoted CG | `account_513076055_campaign_performance_report.csv` | PLATFORM (Fusion) |

**Parse (UTF-16 encoded):**
```python
import csv
with open("devotedstudios 1.csv", encoding='utf-16') as f:
    lines = f.readlines()
for i, l in enumerate(lines):
    if 'Campaign Name' in l: header_i = i; break
reader = csv.DictReader(lines[header_i:], delimiter='\t')
```

**STUDIO May 2026 paid breakdown:**
- Campaign "DS Boosting | $600 monthly": Nethunt BD audience targeting (7 boosted doc posts)
- Campaign "DS | Artstation": ArtStation launch ($150)
- Combined budget: $600 DS + $800 BD personal = $1,400/mo
- May actual: $1,217 (on budget)

**PLATFORM May 2026 paid breakdown:**
- Campaign "AS Partnership" ($2,000 total budget): started May 30, runs June
- May spend: $156 ($39 "Nobody warns" boost + $109 ArtStation launch)
- Budget intentionally saved for June — say "Intentional" not "under budget"

---

## Step 4 — GA4 data

### STUDIO (devotedstudios.com)

Export: GA4 → Reports → Acquisition → Traffic acquisition → Source/medium → Export CSV

File: `Traffic_acquisition_Session_source_medium.csv`

**May 2026 LinkedIn traffic:**
- 114 sessions (8.9% of 1,281 total)
- 91% engagement rate
- 297 key events
- `linkedin/cpc mimic-codev`: 12 sessions @ 104s avg (highest intent)

### PLATFORM (Devoted CG — devotedfusion.com)

Export: same path, different property

File: `Traffic_acquisition_Session_source_medium (1).csv`

**May 2026 LinkedIn traffic:**
- 166 sessions (9.2% of 1,808 total)
- 93% avg engagement rate
- 1,388 key events
- `linkedin.com/referral`: 26 sessions @ 308s avg (5+ min — highest intent)
- `asxfusion1` (ArtStation launch): 24 sessions @ 80s avg
- `post0528` ("Nobody warns"): 16 sessions

**Other notable PLATFORM sources:**
- `artstation.com/referral`: 62 sessions @ 92% eng (confirmed by data)
- `sendgrid/email`: 211 sessions (#2 source — strong owned channel)
- `bd/referral/marina` + `bd/referral/ninel`: BD team UTM tracking is live
- `youtube/social/devotedtalks-amir-satvat`: 6 sessions @ 226s avg

**Parse GA4 CSV:**
```python
with open("Traffic_acquisition_Session_source_medium.csv") as f:
    lines = f.readlines()
for i, l in enumerate(lines):
    if 'Sessions' in l and 'Engaged' in l and not l.startswith('#'):
        header_i = i; break
rows = list(csv.reader(lines[header_i:]))
# row[0]=source, row[1]=campaign, row[2]=sessions, row[3]=engaged,
# row[5]=avg_time, row[8]=key_events
```

---

## Step 5 — Update report HTML

### Scorecard CSS classes

```css
/* Status pills */
.sc-status.green  { background: var(--up-bg);   color: var(--up);   }
.sc-status.yellow { background: var(--warn-bg); color: var(--warn); }
.sc-status.red    { background: var(--down-bg); color: var(--down); }
.sc-status.gray   { background: var(--surface2); color: var(--text3); }

/* Column headers */
.sc-col-header.studio   { background: var(--text); color: #fff; }
.sc-col-header.platform { background: #0891B2; color: #fff; }
/* Ninel = #7C3AED · YouTube = #DC2626 */
```

### Post hyperlink format (always add to notes)

```html
<a href="https://www.linkedin.com/feed/update/urn:li:activity:<URN>/"
   target="_blank" rel="noopener"
   style="color:inherit;text-decoration:underline;text-underline-offset:2px;">Post title</a>
```

For Ninel's personal posts:
```html
<a href="https://www.linkedin.com/posts/ninel-anderson-92674455_<slug>" ...>
```

### KPI targets (from playbook)

| Channel | Metric | Target |
|---------|--------|--------|
| STUDIO LinkedIn | Followers | +500 / mo |
| STUDIO LinkedIn | Free invites | 250 / mo |
| STUDIO LinkedIn | Posts | 15–20 / mo |
| STUDIO LinkedIn | Eng % non-doc | ≥5% |
| STUDIO LinkedIn | Eng % doc carousels | ≥25% |
| STUDIO LinkedIn | Total impressions | ≥40,000 / mo (3-mo avg: Mar 44K · Apr 26K · May 59K) |
| Ninel LinkedIn | Followers | +100 / mo |
| Ninel LinkedIn | Impressions | ≥25,000 / mo |
| Ninel LinkedIn | Originals share | ≥60% |
| Ninel LinkedIn | Eng % | ≥3% |
| PLATFORM LinkedIn | Followers | +30 / mo |
| PLATFORM LinkedIn | Free invites | 250 / mo |
| PLATFORM LinkedIn | Posts | 12–15 / mo |
| PLATFORM LinkedIn | Faves eng % | ≥15% |
| YouTube | Full episodes | 2 / mo |
| YouTube | Shorts | 12 / mo |
| YouTube | Net subscribers | +200 / mo |
| STUDIO Paid CTR | Boosted docs | ≥5% |
| STUDIO Paid budget | Combined | ~$1,400 / mo ($600 DS + $800 BD) |
| PLATFORM Paid budget | LinkedIn | ~$1,000 / mo |

### Historical data

| Month | STUDIO followers | PLATFORM followers | Ninel followers |
|-------|-----------------|-------------------|----------------|
| Mar 2026 end | 19,814 | ~2,098 | 10,882 |
| Apr 2026 end | 20,473 (+659) | ~2,134 (+36) | 10,949 (+67) |
| May 2026 end | 20,941 (+468) | 2,891 (+757) | 11,236 (+287) |
| Jun 8 live | 20,941 | 2,891 | 11,235 |

---

## Step 6 — Ninel top post patterns (repeatable)

From May 2026 analysis:

1. **Game launch posts** → highest reach every time. Name the game + name the partner + personal reaction. Post within 24h of launch.
2. **Personal travel/moments** → #2 every month. Zero product mention. Authentic > polished.
3. **Ninel as PLATFORM amplifier** → Fusion posts by Ninel get 9+ reposts. Her audience (37% game industry, 40% Senior+) = exact PLATFORM buyer. Every major PLATFORM launch needs a Ninel post.
4. **Conference cluster** → announce + live + teaser = 3-post arc, reliably outperforms.
5. **Reshares underperform 2×** → originals avg 3,825 impr vs reshare 1,653 impr. Replace reshare slots with originals.

---

## Step 7 — YouTube data

Pull from YouTube Studio (not playwright-scrape):
- Ask user to screenshot or export YouTube Studio → Analytics → Overview → May 1–31
- Key metrics: Views, Watch time (hours), Subscribers gained

**May 2026 actual:** 3,814 views · 67.2 hrs watch time · +17 subscribers · 0 new episodes · 0 Shorts

---

## Step 8 — Publish to GitHub

```bash
cd "/Users/juls/Documents/playwright-cli-main/SMM REPORT"
git add "social-media-report-2026_8-FINAL.html"
git add "Linkedin Devoted Studios May" 2>/dev/null || true
git status
git commit -m "Add May 2026 SMM report — STUDIO, PLATFORM, Ninel, GA4, Ads"
git push origin main
```

| Field | Value |
|-------|-------|
| Repo | `git@github.com:jplakhotniuk-dev/smm-report.git` |
| Branch | `main` |
| SSH key | `~/.ssh/id_ed25519` |

---

## Known limitations & fixes

- **LinkedIn admin page virtual scroll**: table renders only ~10 rows. Scroll table container: `playwright-cli eval "document.querySelector('table').parentElement.scrollTop = 8000"`
- **Post screenshot wrong element**: find listitem with "Feed post" heading — `grep -B2 "Feed post" "$SNAP" | grep "listitem \[ref="`
- **Persistent profile login**: user-data-dir = `/Users/juls/Library/Caches/ms-playwright/daemon/99d2f184fd7cdb1f/ud-default-chrome`
- **LinkedIn blocks Chrome extension** (Claude in Chrome): use playwright-cli instead
- **CDP not available**: Chrome must be launched with `--remote-debugging-port=9222` — but this requires closing existing Chrome first. Use playwright persistent profile instead.
- **GA4 CSV encoding**: UTF-8, skip comment lines starting with `#`, find header line containing "Sessions,Engaged"
- **LinkedIn Ads CSV encoding**: UTF-16 — open with `encoding='utf-16'`
