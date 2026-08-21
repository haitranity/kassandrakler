---
name: contact-rate-analysis
description: "Run the weekly or monthly Contact Rate (CR) deep-dive: computes specialist-ticket contact rate (7d/WAU weekly, 30d/MAU monthly) and decomposes what drove its movement by CHT category, channel, creation trigger, and incident/known-issue linkage, delivered as a slide-style HTML deck. Use this skill whenever Kass asks to 'run the contact rate report', 'what drove contact rate', 'why did CR move', 'weekly contact rate deep dive', 'monthly contact rate deep dive', or any question about the User Voice Contact Rate metric and its drivers. Always use this skill — not ad-hoc Snowflake queries — because it has the verified metric definition (report_uhe_consolidation for monthly, report_support_tickets_specops for weekly/drivers) and validated queries."
---

Analyse **Contact Rate (CR)** — the number of specialist-handled support tickets over active users — and explain what drove its movement. Deliver a slide-style HTML deck. Supports both cadences; ask Kass which one if not specified (default: monthly, since that's the standing Launch Happiness input metric).

## Metric definitions (verified Aug 2026)

- **Monthly CR** = `specialist_tickets_rolling_30_days / users_active_rolling_30_days` (30d tickets / MAU), read from `ds_prod.report.report_uhe_consolidation` — a pre-aggregated daily-grain dbt table (one row per `calendar_date` × segment combo: `holdout_cohort`, `country_code`, `locale_code`, `platform`, `plan_type`, `plan_is_free`, `is_free_cht`). For the headline (unsegmented), `SUM()` both numerator and denominator across all segment rows per `calendar_date` — the segments exhaustively partition tickets/users, so summing reconstructs the total. Report on `is_last_day_of_month = true` dates. **Confirmed** by pulling the "Monthly Contact Rate" tile off Kass's `scm_airr` Looker dashboard — its field is literally `report_uhe_consolidation.contact_rate`.
- **Weekly CR** = trailing-7-day count of specialist tickets / WAU, for the week ending on the reporting Sunday (Kass's convention: "week beginning 2026-08-10" reads off the row for Sunday 2026-08-16). No pre-aggregated 7-day table exists — build it ticket-level:
  - Numerator: `ds_prod.report.report_support_tickets_specops` — a wide **ticket-grain** table (199 columns; `dim_support_ticket_dkey` is one row per ticket). `COUNT(DISTINCT dim_support_ticket_dkey)` where `created_at::date` falls in the trailing 7 days, filtered by the three conditions below.
  - Denominator (WAU): `ds_prod.report.users_active_v2` — single row per `date_range_upper`, already a global total (no segment columns, no need to SUM across anything). Use `users_active_rolling_7_days`.
- **The three baked-in filters** (confirmed from the actual LookML `explore: report_support_tickets_specops { always_filter: {...} }`, not just inferred): every query against `report_support_tickets_specops` must apply all three to match what Kass sees in Looker —
  - `is_duplicate = false`
  - `does_contribute_to_uv_metrics = true`
  - `is_auto_resolved_successful = false` — this is what makes it "specialist-handled" (excludes Automation), consistent with the Automation vs Specops separation convention.
- **Specialist tickets = "contact"**. The filter above already excludes auto-resolved/automation tickets, so CR is inherently Specops-only. Don't additionally filter out automation — it's already out. If Kass wants the automation-inclusive view (all tickets regardless of resolution path), drop the `is_auto_resolved_successful = false` filter and label it clearly as a different, broader metric.
- **No free/paid or country segment cuts.** Kass has explicitly said these aren't relevant to contact rate reporting — this overrides the general "always segment free/paid" convention for this specific skill. Don't add them back in.

### Verified dimension sources for driver decomposition

All four live directly on `report_support_tickets_specops` (ticket grain, or one join for taxonomy/incidents) — apply the same three filters as above to every query. **Channel and creation_trigger are analysed separately, not cross-tabbed together** — a combined channel×trigger table is confusing and mixes two different questions ("how did it arrive" vs "what triggered it").

- **Channel** (Email / Chat) — `report_support_tickets_specops.channel`. Own contribution ranking, own table.
- **Creation trigger** (the business process that created the ticket, e.g. Help Assistant, Contact Form) — `report_support_tickets_specops.creation_trigger`. Own contribution ranking, own table.
- **CHT taxonomy** — join to the category table (already flat, no array parsing needed): `LEFT JOIN ds_prod.report.report_support_ticket_cht_category c ON t.dim_support_ticket_dkey = c.dim_support_ticket_dkey`.
  - Use **`c.category_level_4`** (most granular — what Kass's Looker explore calls `cht_category`) as the **primary driver dimension** — this is what actually pinpoints where an increase or decrease came from.
  - Use `c.category_level_1` (key_theme) / `c.category_level_2` (main_category) as a secondary **overview** table only — useful context (e.g. "C4E Application is up overall") but not the main point. Don't rank main_category by contribution score as the headline taxonomy finding; drill straight to `category_level_4`.
  - Rank `category_level_4` contributors by contribution score and **split into two separate tables: top 5 increases and top 5 decreases.** Don't mix increases and decreases sorted by absolute value in one table — it reads as confusing since the reader has to scan the sign column to tell which direction each row moved.
  - **A ticket can carry more than one category** (`category_entry_number` > 1 on some rows) — the join fans out, so `COUNT(*)` per category row is fine for "which categories moved" but `COUNT(DISTINCT dim_support_ticket_dkey)` will under-total if you group across categories. Treat category-level counts as diagnostic mix (same caveat pattern as the 60-min resolution skill's ARRAY-flatten node categories), and always cross-check the ticket total against the headline query (Query A).
- **Incidents & known issues** — `report_support_tickets_specops.is_incident` / `is_known_issue`, joined to their driving incident via **`parent_zendesk_ticket_id`** (Looker label "Parent Support Ticket Link"). A single incident or bug often generates a burst of linked tickets, and this is frequently a real driver of CR movement that taxonomy/channel cuts alone won't surface cleanly (they'll just show as a spike in whatever category the incident falls under). Group linked tickets by `parent_zendesk_ticket_id` to count how many child tickets trace back to each incident/bug, curr vs prev period, and surface the top ones by current-period linked ticket count.
- **Contribution ranking (Kass's method)**: rank channel / creation_trigger / CHT category contributors by their **contribution to the CR shift** — `delta_tickets(s) / curr_WAU * 100`, in pp — not by raw volume delta. A category can gain volume but still not be "driving" CR if WAU grew proportionally. See Step 3.

## Step 1: Determine the window

- **Weekly**: default to the most recently completed week (Sun-ending, per Kass's convention). Compute via Bash `date` — do not hardcode. `curr_week_end` = most recent Sunday ≤ today; `prev_week_end` = `curr_week_end - 7`; `trend_start` = `curr_week_end - 12 weeks` (12-week trend).
- **Monthly**: default to **MoM: the last fully completed calendar month vs the month before**, unless Kass specifies otherwise. `curr_month_end` = last day of last completed month; `prev_month_end` = last day of the month before that; `trend_start` = `curr_month_end` − 5 months (6-month trend).
- Ask Kass which cadence if the request doesn't say (per CLAUDE.md: ask before writing SQL).

## Step 2: Run the query bundle via data-mcp

Use `execute_query` + `get_query_result`. Templates below use columns confirmed via `quick_sample` against the live tables (not just dbt docs — see Caveats on why that distinction matters). `{curr_wau}` is the current period's WAU/MAU value pulled from Query A — use it as the fixed denominator for every contribution score so scores across dimensions are additive and comparable.

### Query A (monthly) — Headline trend

```sql
SELECT
  calendar_date,
  SUM(specialist_tickets_rolling_30_days) AS specialist_tickets_30d,
  SUM(users_active_rolling_30_days) AS mau,
  ROUND(100.0 * SUM(specialist_tickets_rolling_30_days) / NULLIF(SUM(users_active_rolling_30_days), 0), 3) AS contact_rate_pct
FROM ds_prod.report.report_uhe_consolidation
WHERE calendar_date >= '{trend_start}' AND calendar_date <= '{curr_month_end}'
GROUP BY 1 ORDER BY 1
-- Purpose: monthly contact rate headline trend
-- keywords: contact rate, specialist tickets, MAU, report_uhe_consolidation
```

### Query A (weekly) — Headline trend

```sql
WITH daily_tickets AS (
  SELECT created_at::date AS ticket_date, dim_support_ticket_dkey
  FROM ds_prod.report.report_support_tickets_specops
  WHERE source = 'zendesk'
    AND is_duplicate = false
    AND does_contribute_to_uv_metrics = true
    AND is_auto_resolved_successful = false
    AND created_at::date BETWEEN '{trend_start}' - 6 AND '{curr_week_end}'
),
daily AS (
  SELECT ticket_date, COUNT(DISTINCT dim_support_ticket_dkey) AS tickets
  FROM daily_tickets
  GROUP BY 1
),
wau AS (
  SELECT date_range_upper AS calendar_date, users_active_rolling_7_days AS wau
  FROM ds_prod.report.users_active_v2
  WHERE date_range_upper BETWEEN '{trend_start}' AND '{curr_week_end}'
)
SELECT
  w.calendar_date,
  SUM(d.tickets) OVER (ORDER BY w.calendar_date RANGE BETWEEN INTERVAL '6 day' PRECEDING AND CURRENT ROW) AS tickets_rolling_7d,
  w.wau,
  ROUND(100.0 * SUM(d.tickets) OVER (ORDER BY w.calendar_date RANGE BETWEEN INTERVAL '6 day' PRECEDING AND CURRENT ROW) / NULLIF(w.wau, 0), 3) AS contact_rate_pct
FROM wau w
LEFT JOIN daily d ON d.ticket_date = w.calendar_date
WHERE w.calendar_date >= '{trend_start}'
ORDER BY 1
-- Purpose: weekly contact rate headline trend (rolling 7d tickets / WAU)
-- keywords: contact rate, specialist tickets, WAU, report_support_tickets_specops, users_active_v2
```

### Query B1 — Channel drivers (curr + prev period, standalone)

```sql
SELECT
  channel,
  COUNT(DISTINCT CASE WHEN created_at::date BETWEEN '{curr_period_start}' AND '{curr_period_end}' THEN dim_support_ticket_dkey END) AS curr_tickets,
  COUNT(DISTINCT CASE WHEN created_at::date BETWEEN '{prev_period_start}' AND '{prev_period_end}' THEN dim_support_ticket_dkey END) AS prev_tickets,
  (curr_tickets - prev_tickets) AS delta_tickets,
  ROUND(100.0 * (curr_tickets - prev_tickets) / {curr_wau}, 5) AS contribution_pp
FROM ds_prod.report.report_support_tickets_specops
WHERE source = 'zendesk'
  AND is_duplicate = false
  AND does_contribute_to_uv_metrics = true
  AND is_auto_resolved_successful = false
  AND created_at::date BETWEEN '{prev_period_start}' AND '{curr_period_end}'
GROUP BY 1 ORDER BY ABS(delta_tickets) DESC
-- Purpose: contact rate channel driver mix (channel only, not cross-tabbed with creation_trigger)
-- keywords: contact rate, channel, report_support_tickets_specops
```

### Query B2 — Creation trigger drivers (curr + prev period, standalone)

Same shape as Query B1, `GROUP BY creation_trigger` instead of `channel`.

### Query C1 — CHT taxonomy overview (main_category / key_theme, context only)

```sql
SELECT
  COALESCE(c.category_level_1, 'Unknown') AS key_theme,
  COALESCE(c.category_level_2, 'Unknown') AS main_category,
  COUNT(DISTINCT CASE WHEN t.created_at::date BETWEEN '{curr_period_start}' AND '{curr_period_end}' THEN t.dim_support_ticket_dkey END) AS curr_tickets,
  COUNT(DISTINCT CASE WHEN t.created_at::date BETWEEN '{prev_period_start}' AND '{prev_period_end}' THEN t.dim_support_ticket_dkey END) AS prev_tickets
FROM ds_prod.report.report_support_tickets_specops t
LEFT JOIN ds_prod.report.report_support_ticket_cht_category c ON t.dim_support_ticket_dkey = c.dim_support_ticket_dkey
WHERE t.source = 'zendesk' AND t.is_duplicate = false AND t.does_contribute_to_uv_metrics = true AND t.is_auto_resolved_successful = false
  AND t.created_at::date BETWEEN '{prev_period_start}' AND '{curr_period_end}'
GROUP BY 1,2 ORDER BY curr_tickets DESC
-- Purpose: CHT taxonomy overview at main_category level — context, not the primary driver finding
-- keywords: contact rate, main_category, key_theme, taxonomy overview
```

### Query C2 — CHT category drivers (curr + prev period, the primary taxonomy finding)

```sql
WITH by_period AS (
  SELECT
    COALESCE(c.category_level_1, 'Unknown') AS key_theme,
    COALESCE(c.category_level_2, 'Unknown') AS main_category,
    COALESCE(c.category_level_4, 'Unknown') AS cht_category,
    COUNT(DISTINCT CASE WHEN t.created_at::date BETWEEN '{curr_period_start}' AND '{curr_period_end}' THEN t.dim_support_ticket_dkey END) AS curr_tickets,
    COUNT(DISTINCT CASE WHEN t.created_at::date BETWEEN '{prev_period_start}' AND '{prev_period_end}' THEN t.dim_support_ticket_dkey END) AS prev_tickets
  FROM ds_prod.report.report_support_tickets_specops t
  LEFT JOIN ds_prod.report.report_support_ticket_cht_category c ON t.dim_support_ticket_dkey = c.dim_support_ticket_dkey
  WHERE t.source = 'zendesk' AND t.is_duplicate = false AND t.does_contribute_to_uv_metrics = true AND t.is_auto_resolved_successful = false
    AND t.created_at::date BETWEEN '{prev_period_start}' AND '{curr_period_end}'
  GROUP BY 1,2,3
)
SELECT *, (curr_tickets - prev_tickets) AS delta_tickets, ROUND(100.0 * (curr_tickets - prev_tickets) / {curr_wau}, 5) AS contribution_pp
FROM by_period
ORDER BY delta_tickets DESC
-- Purpose: CHT category (most granular) contribution ranking. Take the top 5 rows (ORDER BY delta_tickets DESC) for
-- the "top 5 increases" table, and the bottom 5 rows (ORDER BY delta_tickets ASC) for "top 5 decreases" —
-- present as two separate tables, not one table mixing both directions.
-- keywords: contact rate, cht_category, category_level_4, taxonomy driver, contribution score
```

### Query E1 — Incidents & known issues overview (curr + prev period)

```sql
SELECT
  COUNT(DISTINCT CASE WHEN created_at::date BETWEEN '{curr_period_start}' AND '{curr_period_end}' AND is_incident THEN dim_support_ticket_dkey END) AS curr_incident_tickets,
  COUNT(DISTINCT CASE WHEN created_at::date BETWEEN '{prev_period_start}' AND '{prev_period_end}' AND is_incident THEN dim_support_ticket_dkey END) AS prev_incident_tickets,
  COUNT(DISTINCT CASE WHEN created_at::date BETWEEN '{curr_period_start}' AND '{curr_period_end}' AND is_known_issue THEN dim_support_ticket_dkey END) AS curr_known_issue_tickets,
  COUNT(DISTINCT CASE WHEN created_at::date BETWEEN '{prev_period_start}' AND '{prev_period_end}' AND is_known_issue THEN dim_support_ticket_dkey END) AS prev_known_issue_tickets,
  COUNT(DISTINCT CASE WHEN created_at::date BETWEEN '{curr_period_start}' AND '{curr_period_end}' THEN dim_support_ticket_dkey END) AS curr_total_tickets,
  COUNT(DISTINCT CASE WHEN created_at::date BETWEEN '{prev_period_start}' AND '{prev_period_end}' THEN dim_support_ticket_dkey END) AS prev_total_tickets
FROM ds_prod.report.report_support_tickets_specops
WHERE source = 'zendesk' AND is_duplicate = false AND does_contribute_to_uv_metrics = true AND is_auto_resolved_successful = false
  AND created_at::date BETWEEN '{prev_period_start}' AND '{curr_period_end}'
-- Purpose: what share of contact volume is incident/known-issue driven, and did that share move
-- keywords: contact rate, is_incident, is_known_issue, incident driven contacts
```

### Query E2 — Top individual incidents/known issues by linked ticket count

```sql
SELECT
  parent_zendesk_ticket_id,
  MAX(is_incident::int) = 1 AS is_incident,
  MAX(is_known_issue::int) = 1 AS is_known_issue,
  COUNT(DISTINCT CASE WHEN created_at::date BETWEEN '{curr_period_start}' AND '{curr_period_end}' THEN dim_support_ticket_dkey END) AS curr_linked_tickets,
  COUNT(DISTINCT CASE WHEN created_at::date BETWEEN '{prev_period_start}' AND '{prev_period_end}' THEN dim_support_ticket_dkey END) AS prev_linked_tickets
FROM ds_prod.report.report_support_tickets_specops
WHERE source = 'zendesk' AND is_duplicate = false AND does_contribute_to_uv_metrics = true AND is_auto_resolved_successful = false
  AND (is_incident = true OR is_known_issue = true)
  AND parent_zendesk_ticket_id IS NOT NULL
  AND created_at::date BETWEEN '{prev_period_start}' AND '{curr_period_end}'
GROUP BY 1
ORDER BY curr_linked_tickets DESC
LIMIT 15
-- Purpose: pinpoint which specific incident/bug (by parent Zendesk ticket) is generating the most linked tickets this period
-- keywords: contact rate, parent_zendesk_ticket_id, incident, known issue, linked tickets
```

## Step 3: Decompose "what drove it"

**Headline split (ticket-volume effect vs WAU effect)** — same as before, do this arithmetic in Python via Bash:
- `ticket_effect = (curr_tickets - prev_tickets) / curr_WAU * 100`
- `wau_effect = prev_tickets * (1/curr_WAU - 1/prev_WAU) * 100`
- `ticket_effect + wau_effect ≈ headline Δ` (small residual is fine)

**Per-dimension contribution scores** — for channel (Query B1), creation_trigger (Query B2), and CHT category (Query C2), each **independently**:
- `contribution_pp(s) = (curr_tickets(s) − prev_tickets(s)) / curr_WAU × 100`
- Rank by `|contribution_pp(s)|` for channel and creation_trigger (small number of segments, one table each is fine).
- For CHT category, split into **top 5 increases** (`contribution_pp > 0`, sorted descending) and **top 5 decreases** (`contribution_pp < 0`, sorted ascending) as **two separate tables**.
- Sum of all category-level contributions ≈ `ticket_effect` from the headline split (small residual from long-tail categories is fine; note it if large).
- Show the main_category/key_theme overview (Query C1) as **supporting context only** — e.g. "C4E Application is up overall" — but lead the narrative with the `category_level_4` findings, since that's what actually pinpoints the issue.

**Incidents & known issues**:
- From Query E1: report the incident/known-issue share of total contact volume, curr vs prev, and call out if that share moved meaningfully.
- From Query E2: list the top parent tickets by `curr_linked_tickets`, flagging any that are new this period (`prev_linked_tickets = 0`) or growing fast — these are often the cleanest, most actionable "why did CR move" answer (a single bug or incident, not a diffuse taxonomy shift).

Report contributions in **pp** (percentage points) to enough decimal places to be non-zero (CR itself is a small percentage here, so 2-3 decimals will round pp contributions to 0.00 — use more precision, e.g. 5 decimals, and say so). Always pair relative and absolute (e.g. "+810 tickets (+11%), +0.00087pp").

## Step 4: Build the deck

Produce a single self-contained HTML file (slide-style sections, no external JS; simple CSS/SVG bar charts), saved to the outputs folder, named `contact-rate-{weekly|monthly}-{period}.html`. Structure:

1. **Title / headline** — current-period CR, period-over-period Δ in pp, trend sparkline (12 weeks or 6 months)
2. **What drove it** — ticket-volume effect vs WAU effect waterfall
3. **Channel drivers** — standalone table, ranked by |contribution|
4. **Creation trigger drivers** — standalone table, ranked by |contribution|
5. **CHT taxonomy** — main_category/key_theme overview table (context), then **top 5 increases** and **top 5 decreases** at `category_level_4`, as two separate tables
6. **Incidents & known issues** — overview stats, then top individual incidents/bugs by linked ticket count (curr vs prev)
7. **Appendix** — metric definition, window, caveats, and the SQL used

No segment-cuts slide (free/paid, country) — not relevant to this report per Kass's guidance.

Formatting: rates to 2-3 decimals, contributions in pp to enough decimals to be legible (5 is usually right for this metric), counts with thousands commas, ↑/↓ arrows for direction. Present the file to Kass with a 3-4 bullet summary in chat.

## Caveats

- **Never trust `describe_table_or_view` as the full schema for these SCM/UV report tables.** It only returns dbt's documented column subset (this bit us once: it showed `report_support_tickets_specops` as 20 DNE/FCR columns when the live table actually has 199, including `channel`, `creation_trigger`, and the category arrays). Always confirm real columns with `quick_sample` before writing a query against a new table in this space, especially anything under `ds_prod.report.*`.
- Always apply **all three** baked-in filters together on `report_support_tickets_specops` (`is_duplicate = false`, `does_contribute_to_uv_metrics = true`, `is_auto_resolved_successful = false`) — dropping any one will silently overcount vs what Kass sees in Looker.
- CHT taxonomy join fans out for multi-category tickets — `COUNT(*)` per category is mix diagnostic only; use `COUNT(DISTINCT dim_support_ticket_dkey)` for actual ticket totals and reconcile against Query B1/A for the same period before trusting the taxonomy split.
- `parent_zendesk_ticket_id` is typically **null on the root/first ticket of an incident** and only populated on the follow-on tickets linked to it — so `curr_linked_tickets` in Query E2 counts *additional* tickets an incident generated, not the incident's own root ticket. Don't present it as "total tickets for this incident" without noting the +1 for the root.
- Contact rate as defined here is **already Specops-only** (excludes auto-resolved/automation tickets by construction). Don't re-apply an Automation vs Specops filter on top of it — it's already applied. If a broader "all tickets regardless of resolution path" view is wanted, say so explicitly and drop the `is_auto_resolved_successful` filter, labelled as a different metric.
- `ds_prod.report.users_active` (no suffix) is a **broken view** (missing staging dependency) — always use `users_active_v2`, which is confirmed live and single-grain (one row per date, no segment summing needed).
- **No free/paid or country segment cuts** — Kass has said explicitly these aren't relevant to contact rate reporting. Don't add them back in even though the general standing convention says to segment free/paid for engagement metrics; this skill is an intentional exception.
- Report volumes as absolute + relative always: "1,200 tickets (+8%)", not just one or the other.
- `category_level_1`/`category_level_4` = 'Unknown' (or null) → keep in the table, don't lead the narrative with it.
- If any query returns null/zero unexpectedly, stop and tell Kass rather than shipping a partial deck.
