---
name: kpi-bot
version: 1.0.0
description: "Builds a daily Slack KPI bot: takes any metric + SQL query, creates an n8n workflow that posts daily updates to a Slack channel"
disable-model-invocation: true
---
Build a daily KPI bot that sends a formatted Slack message with data from Databricks. The user will describe what data they want — your job is to write the SQL, format it for Slack, create or update an n8n workflow, and send a test message.

---

## Step 1: Understand the Request

Ask the user (if not already clear from $ARGUMENTS):
1. **What data do you want?** (e.g., "daily bookings by category", "live coach count", "weekly signups")
2. **Which Slack channel?** (get the channel ID — they can find it by right-clicking the channel → View channel details → copy the ID at the bottom)
3. **What time should it send?** (default: 8pm MT)

Remind them: the bot must be invited to the channel first — `/invite @LeLoop` in the target channel.

---

## Step 2: Write the SQL

Query `leland_analytics.ai` tables by default. Key tables and columns:

**bookings** (approved orders only):
- `approved_at_pt` (timestamp) — use for date filtering
- `amount_booked` (USD) — cash revenue (use this, NOT `general_manager_bookings`, to match team dashboards)
- `category_slug` (text) — coaching category (mba, medical, law, etc.). Already a column — no join needed
- `order_type` (text) — hourlyOrder, packageOrder, classOrder, bootcampOrder, subscriptionOrder, contentOrder
- `order_status` — filter: `IN ('APPROVED_BY_COACH', 'COACHING_COMPLETED')`
- Credit columns: `credit_bootcamp_used`, `credit_gift_card_used`, `credit_marketing_giveaway_used`, `credit_partnership_used`, `credit_referral_used`, `credit_refund_used`, `credit_trial_used`, `credit_unknown_used`
- GMV = `amount_booked` + all credit columns

**coachmetricsviews**: Coach performance. `coach_status = 'LIVE'` for live coaches.

**weekly_active_users** / **monthly_active_users**: WAU/MAU metrics. Columns: `week_start`/`month_start`, `wau`/`mau`.

**events**: Leland events with RSVP/attendee counts.

**CRITICAL SQL RULES:**
- Use `category_slug`, never `category`
- Use `amount_booked`, never `revenue`
- Use `approved_at_pt` for dates
- The bookings table already filters to approved orders — no need for `order_status` filter unless you want COACHING_COMPLETED too
- Only use tables/columns listed above or in `leland_analytics.ai`

Test the SQL via Databricks before proceeding:
```python
# Use the Databricks connection pattern from CLAUDE.md
result = run_sql(your_sql)
```

---

## Step 3: Format for Slack

Write JavaScript that formats query results into a readable Slack message using mrkdwn. Patterns that work well:

- **Header:** `:chart_with_upwards_trend: *Bot Name*` with emoji
- **Summary stats:** `*Total:* 1,234` on separate lines
- **Tables:** Use code blocks with aligned columns:
```
```Category   | Booked   | Target   | Pace
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MBA          |   $45.2K |   $60.0K |  112%```
```
- **Pacing %**: Normalize for day of month: `pacingPct = (rawPct / expectedPct) * 100` where `expectedPct = (dayOfMonth / daysInMonth) * 100`
- **Slack gotcha:** The header text MUST be on the same line as the opening triple-backtick, or Slack swallows it

---

## Step 4: Create/Update n8n Workflow

Use the n8n API to create a workflow with these nodes:
1. **Schedule Trigger** — cron at the requested time, timezone `America/Denver`
2. **Execute Workflow** — calls Databricks sub-workflow `v8CeBNa4Bp90sbXw1YVx0` with `{ "query": "YOUR SQL" }`
3. **Code node** — JavaScript formatter from Step 3
4. **HTTP Request** — POST to `https://slack.com/api/chat.postMessage` with LeLoop bot token

**n8n API details:**
- Host: `https://lelandteam.app.n8n.cloud`
- API key: check memory file `email-pipeline.md` for current key
- LeLoop bot token: check memory file `email-pipeline.md`
- Create: `POST /api/v1/workflows` with `{ name, nodes, connections, settings }`
- Activate: `POST /api/v1/workflows/{id}/activate` (separate call — `active` is read-only on PUT)
- `settings: { executionOrder: "v1" }`

**n8n API gotchas:**
- Cannot assign credentials via API — Execute Workflow and HTTP Request nodes don't need credentials
- The Databricks sub-workflow handles all auth internally
- Always use `POST /activate` endpoint, never set `active: true` in PUT body

---

## Step 5: Send Test Message

Run the SQL query via Databricks, format it with the JS logic (replicated in Python), and post directly to the Slack channel to verify everything looks right. Use the LeLoop bot token from memory.

---

## Self-Serve Alternative

There's also a **KPI Bot Factory** form (n8n workflow `bkYcMw31Hhrqfve2`) that lets anyone fill out a form to create a bot without Claude Code. But it requires the Anthropic credential to be assigned in the n8n UI. For Claude Code users, this skill is the more reliable path.

---

**The request:**

$ARGUMENTS
