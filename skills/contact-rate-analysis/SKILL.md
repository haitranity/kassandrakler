---
name: contact-rate-analysis
description: "Run the weekly or monthly Contact Rate (CR) deep-dive: computes specialist-ticket contact rate (7d/WAU weekly, 30d/MAU monthly) and decomposes what drove its movement by CHT taxonomy and channel/entry point, delivered as a slide-style HTML deck. Use this skill whenever Kass asks to 'run the contact rate report', 'what drove contact rate', 'why did CR move', 'weekly contact rate deep dive', 'monthly contact rate deep dive', or any question about the User Voice Contact Rate metric and its drivers. Always use this skill — not ad-hoc Snowflake queries — because it has the verified metric definition (report_uhe_consolidation for monthly, report_support_tickets_specops for weekly/drivers) and validated queries."
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

### Verified dimension sources for driver decomposition

Both live directly on `report_support_tickets_specops` (ticket grain) — apply the same three filters as above to every query.

- **Channel / entry point** — no join needed:
  - `report_support_tickets_specops.channel` (Email / Chat)
  - `report_support_tickets_specops.creation_trigger` (the business process that created the ticket, e.g. Help Assistant, Contact Form)
- **CHT taxonomy** — join to the category table (already flat, no array parsing needed):
  - `LEFT JOIN ds_prod.report.report_support_ticket_cht_category c ON t.dim_support_ticket_dkey = c.dim_support_ticket_dkey`
  - Use `c.category_level_1` (key_theme), `c.category_level_2` (main_category), `c.category_level_3` (user_issue), `c.category_level_4` (most granular — what Kass's Looker explore calls `cht_category`), `c.moment`
  - **A ticket can carry more than one category** (`category_entry_number` > 1 on some rows) — the join fans out, so `SUM(1)` per category row is fine for "which categories moved" but `COUNT(DISTINCT dim_support_ticket_dkey)` will under-total if you group across categories. Treat category-level counts as diagnostic mix (same caveat pattern as the 60-min resolution skill's ARRAY-flatten node categories), and always cross-check the ticket total against the headline query (Query A).
- **Contribution ranking (Kass's method)**: rank taxonomy/channel segments by their **contribution to the CR shift** (rate × mix decomposition, see Step 3 below) — not by raw volume delta. A category can gain volume but still not be "driving" CR if MAU/WAU grew proportionally.

## Step 1: Determine the window

- **Weekly**: default to the most recently completed week (Sun-ending, per Kass's convention). Compute via Bash `date` — do not hardcode. `curr_week_end` = most recent Sunday ≤ today; `prev_week_end` = `curr_week_end - 7`; `trend_start` = `curr_week_end - 12 weeks` (12-week trend).
- **Monthly**: default to **MoM: the last fully completed calendar month vs the month before**, unless Kass specifies otherwise. `curr_month_end` = last day of last completed month; `prev_month_end` = last day of the month before that; `trend_start` = `curr_month_end` − 5 months (6-month trend).
- Ask Kass which cadence if the request doesn't say (per CLAUDE.md: ask before writing SQL).

## Step 2: Run the query bundle via data-mcp

Use `execute_query` + `get_query_result`. Templates below use columns confirmed via `quick_sample` against the live tables (not just dbt docs — see Caveats on why that distinction matters). Run the monthly OR weekly bundle depending on cadence.

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

### Query B — Channel / entry point drivers (curr + prev period)

```sql
SELECT
  created_at::date AS ticket_date,
  channel,
  creation_trigger,
  COUNT(DISTINCT dim_support_ticket_dkey) AS tickets
FROM ds_prod.report.report_support_tickets_specops
WHERE source = 'zendesk'
  AND is_duplicate = false
  AND does_contribute_to_uv_metrics = true
  AND is_auto_resolved_successful = false
  AND created_at::date BETWEEN '{prev_period_start}' AND '{curr_period_end}'
GROUP BY 1,2,3 ORDER BY 1, tickets DESC
-- Purpose: contact rate channel / creation_trigger driver mix
-- keywords: contact rate, channel, creation_trigger, report_support_tickets_specops
```

### Query C — CHT taxonomy drivers (curr + prev period)

```sql
SELECT
  t.created_at::date AS ticket_date,
  COALESCE(c.category_level_1, 'Unknown') AS key_theme,
  COALESCE(c.category_level_2, 'Unknown') AS main_category,
  COALESCE(c.category_level_4, 'Unknown') AS cht_category,
  COUNT(*) AS category_tagged_tickets,          -- diagnostic mix, not a ticket total (see caveats: tickets can carry >1 category)
  COUNT(DISTINCT t.dim_support_ticket_dkey) AS distinct_tickets
FROM ds_prod.report.report_support_tickets_specops t
LEFT JOIN ds_prod.report.report_support_ticket_cht_category c
  ON t.dim_support_ticket_dkey = c.dim_support_ticket_dkey
WHERE t.source = 'zendesk'
  AND t.is_duplicate = false
  AND t.does_contribute_to_uv_metrics = true
  AND t.is_auto_resolved_successful = false
  AND t.created_at::date BETWEEN '{prev_period_start}' AND '{curr_period_end}'
GROUP BY 1,2,3,4 ORDER BY 1, category_tagged_tickets DESC
-- Purpose: contact rate CHT taxonomy driver decomposition
-- keywords: contact rate, CHT category, report_support_ticket_cht_category, taxonomy
```

### Query D — Free / paid + country segment cuts (appendix, curr + prev period)

Per standing convention: always segment free vs paid for engagement metrics. Monthly uses `report_uhe_consolidation`; weekly uses `report_support_tickets_specops.is_free` / `country_code` directly with the same three filters as Query B.

```sql
SELECT
  calendar_date,
  plan_is_free,
  SUM(specialist_tickets_rolling_30_days) AS specialist_tickets_30d,
  SUM(users_active_rolling_30_days) AS mau,
  ROUND(100.0 * SUM(specialist_tickets_rolling_30_days) / NULLIF(SUM(users_active_rolling_30_days), 0), 3) AS contact_rate_pct
FROM ds_prod.report.report_uhe_consolidation
WHERE calendar_date IN ('{prev_period_end}', '{curr_period_end}')
GROUP BY 1,2 ORDER BY 1, specialist_tickets_30d DESC
-- Purpose: contact rate free/paid segment cut (monthly)
-- keywords: contact rate, plan_is_free, free paid segmentation
```

## Step 3: Decompose "what drove it"

For each driver dimension (channel, creation_trigger, CHT taxonomy at whichever level has the most legible number of segments — usually `main_category`, drilling to `cht_category` for the top 3-5), decompose the period-over-period headline change into rate vs mix effects per segment *s*, exactly as the 60-min resolution skill does:

- `rate_effect(s) = prev_share(s) × (curr_rate(s) − prev_rate(s))`
- `mix_effect(s) = (curr_share(s) − prev_share(s)) × curr_rate(s)`
- Sum of all effects ≈ headline Δ (small residual is fine; note it if it's large relative to the headline move)

Do this arithmetic in Python via Bash, not mentally. Rank segments by `|rate_effect(s)| + |mix_effect(s)|` (Kass's "contribution score" — this is what she means by ranking contributors rather than raw volume delta) and lead the narrative with the top 3-5. Report contributions in **pp** (percentage points), per standing convention — CR moves are small, so translate to pp and to raw ticket counts so the story feels concrete (e.g. "+0.04pp, ~1,400 extra tickets (+6%)" — always pair relative and absolute per standing convention).

## Step 4: Build the deck

Produce a single self-contained HTML file (slide-style sections, no external JS; simple CSS/SVG bar charts), saved to the outputs folder, named `contact-rate-{weekly|monthly}-{period}.html`. Structure:

1. **Title / headline** — current-period CR, period-over-period Δ in pp, trend sparkline (12 weeks or 6 months)
2. **What drove it** — waterfall-style list of top contributions (rate vs mix labelled), ranked by contribution score
3. **Channel / entry point** — channel × creation_trigger table with period-over-period arrows
4. **CHT taxonomy drivers** — top categories by contribution, with volume and rate, drilling from `main_category` to `cht_category` for the top movers
5. **Segment cuts** — free/paid and top-10 country tables
6. **Appendix** — metric definition, window, caveats, and the SQL used

Formatting: rates to 2-3 decimals, contributions in pp to 2-3 decimals, counts with thousands commas, ↑/↓ arrows for direction. Present the file to Kass with a 3-4 bullet summary in chat.

## Caveats

- **Never trust `describe_table_or_view` as the full schema for these SCM/UV report tables.** It only returns dbt's documented column subset (this bit us once: it showed `report_support_tickets_specops` as 20 DNE/FCR columns when the live table actually has 199, including `channel`, `creation_trigger`, and the category arrays). Always confirm real columns with `quick_sample` before writing a query against a new table in this space, especially anything under `ds_prod.report.*`.
- Always apply **all three** baked-in filters together on `report_support_tickets_specops` (`is_duplicate = false`, `does_contribute_to_uv_metrics = true`, `is_auto_resolved_successful = false`) — dropping any one will silently overcount vs what Kass sees in Looker.
- CHT taxonomy join fans out for multi-category tickets — `COUNT(*)` per category is mix diagnostic only; use `COUNT(DISTINCT dim_support_ticket_dkey)` for actual ticket totals and reconcile against Query B/A for the same period before trusting the taxonomy split.
- Contact rate as defined here is **already Specops-only** (excludes auto-resolved/automation tickets by construction). Don't re-apply an Automation vs Specops filter on top of it — it's already applied. If a broader "all tickets regardless of resolution path" view is wanted, say so explicitly and drop the `is_auto_resolved_successful` filter, labelled as a different metric.
- `ds_prod.report.users_active` (no suffix) is a **broken view** (missing staging dependency) — always use `users_active_v2`, which is confirmed live and single-grain (one row per date, no segment summing needed).
- Always segment free vs paid for CR/engagement comparisons per standing convention — don't skip Query D even if not asked, unless Kass says otherwise.
- Report volumes as absolute + relative always: "1,200 tickets (+8%)", not just one or the other.
- `category_level_1` = 'Unknown' (or category fields null) → keep in the table, don't lead the narrative with it.
- If any query returns null/zero unexpectedly, stop and tell Kass rather than shipping a partial deck.
