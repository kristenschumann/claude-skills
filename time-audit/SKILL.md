---
name: time-audit
version: 1.0.0
description: "Weekly personal time audit using macOS Screen Time and Chrome history, outputs an HTML visualization"
disable-model-invocation: true
---
# /time-audit — Weekly Time Audit

Run a personal time audit using macOS Screen Time data and Chrome history. Produces an HTML visualization showing where time actually went and how to improve.

## Instructions

### Step 0: Pre-flight check

Run these in parallel:
1. Verify Full Disk Access: `sqlite3 ~/Library/Application\ Support/Knowledge/knowledgeC.db ".tables" 2>&1`
2. Check system timezone: `date`
3. Find all Chrome History files: `find ~/Library/Application\ Support/Google/Chrome/ -name "History" 2>/dev/null`

If knowledgeC.db returns a permissions error, stop and tell the user:
> "Terminal needs Full Disk Access to read Screen Time data. Go to System Settings → Privacy & Security → Full Disk Access → enable Terminal. Then re-run /time-audit."

### Step 1: Get context (REQUIRED — do not skip)

Before pulling any data, ask the user these questions in a single message:

> **Quick context before I pull the data:**
> 1. What's your role and what does a good week look like for you? (e.g. "GM of Courses — good week = 3+ hours of deep work daily, Slack under control")
> 2. What are you hoping to learn or fix? (e.g. "I think I'm spending too much time reactive", "I want to see if weekends are bleeding into work", "just curious")
> 3. Any apps or sites you already know are time drains you want to flag?

Wait for the user's answers before proceeding. These answers frame all findings.

### Step 2: Pull knowledgeC data

Run in parallel:

**Query A — Weekly app totals:**
```sql
SELECT
  ZOBJECT.ZVALUESTRING as app,
  ROUND(SUM(ZOBJECT.ZENDDATE - ZOBJECT.ZSTARTDATE) / 3600.0, 2) as total_hours,
  COUNT(*) as sessions
FROM ZOBJECT
WHERE ZOBJECT.ZSTREAMNAME = '/app/usage'
  AND ZOBJECT.ZSTARTDATE > (strftime('%s', 'now') - 978307200 - 7*86400)
  AND ZOBJECT.ZVALUESTRING IS NOT NULL
GROUP BY app
ORDER BY total_hours DESC;
```

**Query B — Daily totals with correct day labels:**
```sql
SELECT
  date(ZOBJECT.ZSTARTDATE + 978307200, 'unixepoch', 'localtime') as day,
  strftime('%A', ZOBJECT.ZSTARTDATE + 978307200, 'unixepoch', 'localtime') as weekday,
  ROUND(SUM(ZOBJECT.ZENDDATE - ZOBJECT.ZSTARTDATE) / 3600.0, 2) as total_hours
FROM ZOBJECT
WHERE ZOBJECT.ZSTREAMNAME = '/app/usage'
  AND ZOBJECT.ZSTARTDATE > (strftime('%s', 'now') - 978307200 - 7*86400)
GROUP BY day
ORDER BY day DESC;
```

Note: knowledgeC timestamps use CoreData epoch (Jan 1, 2001). Add 978307200 to convert to Unix. Use `localtime` — never hard-code a UTC offset.

### Step 3: Pull Chrome history

**Find the active profiles:**
Copy each History file to /tmp, then compare MAX(last_visit_time) across all profiles. Only query profiles with data from the last 7 days.

```bash
for profile in "Default" "Profile 1" "Profile 2" "Profile 3" "Profile 4" "Profile 5" "Profile 6"; do
  db="/Users/$(whoami)/Library/Application Support/Google/Chrome/$profile/History"
  cp "$db" "/tmp/chrome_${profile// /_}.db" 2>/dev/null
  result=$(sqlite3 "/tmp/chrome_${profile// /_}.db" "SELECT datetime(MAX(last_visit_time)/1000000 - 11644473600, 'unixepoch', 'localtime') FROM urls;" 2>/dev/null)
  name=$(cat "/Users/$(whoami)/Library/Application Support/Google/Chrome/$profile/Preferences" 2>/dev/null | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('account_info',[{}])[0].get('email','unknown'))" 2>/dev/null)
  echo "$profile ($name): $result"
done
```

For each active profile (last visit within 7 days), run the site categorization query. Adapt the CASE statement categories to match the user's role and tools — add/remove categories based on the context gathered in Step 1.

Chrome timestamps: `last_visit_time / 1000000 - 11644473600` = Unix timestamp. Chrome has visit counts, not time — treat as activity proxy.

**Standard category template:**
```sql
SELECT
  CASE
    WHEN url LIKE '%mail.google.com%' THEN 'Gmail'
    WHEN url LIKE '%docs.google.com%' THEN 'Google Docs/Sheets'
    WHEN url LIKE '%drive.google.com%' THEN 'Google Drive'
    WHEN url LIKE '%calendar.google.com%' THEN 'Google Calendar'
    WHEN url LIKE '%meet.google.com%' THEN 'Google Meet'
    WHEN url LIKE '%notion.so%' OR url LIKE '%notion.com%' THEN 'Notion'
    WHEN url LIKE '%slack.com%' THEN 'Slack (web)'
    WHEN url LIKE '%linkedin.com%' THEN 'LinkedIn'
    WHEN url LIKE '%youtube.com%' THEN 'YouTube'
    WHEN url LIKE '%claude.ai%' OR url LIKE '%anthropic.com%' THEN 'Claude.ai'
    WHEN url LIKE '%github.com%' THEN 'GitHub'
    WHEN url LIKE '%zoom.us%' THEN 'Zoom'
    WHEN url LIKE '%localhost%' OR url LIKE '%127.0.0.1%' THEN 'Localhost (dev)'
    ELSE 'Other'
  END as category,
  COUNT(*) as visits,
  date(last_visit_time/1000000 - 11644473600, 'unixepoch', 'localtime') as day
FROM urls
WHERE last_visit_time/1000000 - 11644473600 > strftime('%s','now') - 7*86400
GROUP BY category, day
ORDER BY day DESC, visits DESC;
```

Also pull top domains in "Other" to identify unlabeled sinks.

### Step 4: Generate HTML visualization

Save to a sensible path — suggest `~/Desktop/time-audit-{month}{year}.html` or wherever the user typically works.

The HTML should include:
1. **4 stat cards** — Total screen time, Deep work hours (terminal/focused tools), top comms app sessions, weekend hours
2. **Daily bar chart** — Color-coded: work days (dark green), personal days (red), mixed (amber), no data (gray), partial (light green). Use actual day names from data — never assign manually.
3. **App totals** — Horizontal bar chart, all apps from knowledgeC
4. **Chrome breakdown** — Work profile vs personal profile side by side, with visit bars
5. **Where work time goes** — Ranked estimated hours per activity
6. **Findings** — 4–6 cards, color-coded green/yellow/red, framed against the user's stated goals from Step 1

Design: clean, minimal, system font, white cards on light gray background. No dependencies — pure HTML/CSS.

Open the file automatically after writing: `open <path>`

### Step 5: Deliver findings

Frame every finding against what the user said in Step 1. Lead with the most surprising or actionable insight, not a summary of what the data shows.

Structure:
1. The one thing that stands out most (1 sentence)
2. 3–4 specific findings, each tied to the user's stated goals
3. 2–3 concrete recommendations — specific and behavioral, not generic ("batch Slack to top of hour" not "use Slack less")

### Notes

- Always derive day-of-week from the date — never assign manually. Use `strftime('%A')` in SQL or Python's `date.strftime('%A')`.
- Chrome history may be locked if Chrome is open. Always `cp` to /tmp first.
- knowledgeC session count is often more revealing than total time — flag high session counts (200+/day on a single app) as fragmentation signals.
- If a profile has a personal Google account, label it clearly and call out that personal browsing will be visible — let the user decide how much detail they want.
- The HTML output is self-contained and shareable — mention this to the user.
