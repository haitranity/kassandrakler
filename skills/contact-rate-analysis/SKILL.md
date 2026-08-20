---
name: contact-rate-analysis
description: "Run the weekly or monthly Contact Rate (CR) deep-dive: computes specialist-ticket contact rate (7d/WAU weekly, 30d/MAU monthly) and decomposes what drove its movement by CHT taxonomy and channel/entry point, delivered as a slide-style HTML deck. Use this skill whenever Kass asks to 'run the contact rate report', 'what drove contact rate', 'why did CR move', 'weekly contact rate deep dive', 'monthly contact rate deep dive', or any question about the User Voice Contact Rate metric and its drivers. Always use this skill — not ad-hoc Snowflake queries — because it has the verified metric definition (report_uhe_consolidation / report_scm_consolidation) and validated driver queries."
---

Analyse **Contact Rate (CR)** — the number of specialist-handled support tickets over active users — and explain what drove its movement. Deliver a slide-style HTML deck. Supports both cadences; ask Kass which one if not specified (default: monthly, since that's the standing Launch Happiness input metric).

## Metric definitions (verified Aug 2026)

- **Monthly CR** = `specialist_tickets_rolling_30_days / users_active_rolling_30_days` (30d tickets / MAU), read from `ds_prod.report.report_uhe_consolidation` — a pre-aggregated daily-grain dbt table (one row per `calendar_date` × segment combo: `holdout_cohort`, `country_code`, `locale_code`, `platform`, `plan_type`, `plan_is_free`, `is_free_cht`). For the headline (unsegmented), `SUM()` both numerator and denominator across all segment rows per `calendar_date` — the segments exhaustively partition tickets/users, so summing reconstructs the total. Report on `is_last_day_of_month = true` dates (mirrors the Looker "Monthly Contact Rate" tile on `user_voice::scm_airr`).
- **Weekly CR** = trailing-7-day sum of specialist tickets / WAU, for the week ending on the reporting Sunday (Kass's convention: "week beginning 2026-08-10" reads off the row for Sunday 2026-08-16). No pre-aggregated 7-day table exists — build it from daily grain:
  - Numerator: `ds_prod.report.report_scm_consolidation` filtered `source = 'zendesk' AND NOT is_auto_resolved`, summed `number_of_support_tickets_created` over the trailing 7 days ending on the reporting date. This is the exact ticket-level filter dbt uses to build `specialist_tickets_rolling_30_days` in `report_uhe_consolidation` (confirmed from the model SQL) — `NOT is_auto_resolved` is what makes it "specialist-handled" i.e. excludes Automation, consistent with the Automation vs Specops separation convention.
  - Denominator (WAU): `ds_prod.report.active_user_daily_with_primary_brand_and_mapu_segments`, `SUM(users_active_rolling_7_days)` filtered `is_internal_user = false`, trailing 7 days ending on the reporting date. **Not yet 100% verified against the Looker WAU number** — sanity-check the first output against the Looker explore before trusting it (see Caveats).
- **Specialist tickets = "contact"**. This already excludes auto-resolved/automation tickets by construction (`NOT is_auto_resolved`), so CR is inherently a Specops-only metric. Don't additionally filter out automation — it's already out. If Kass wants the automation-inclusive view (total ticket volume regardless of resolution path), use `total_support_tickets_created_rolling_30_days` on `report_uhe_consolidation` or the unfiltered `number_of_support_tickets_created` on `report_scm_consolidation` instead, and label it clearly as a different, broader metric.

### Verified dimension sources for driver decomposition

- **Channel / entry point** — already on the daily ticket-count table, no join needed:
  - `report_scm_consolidation.channel` (Email / Chat)
  - `report_scm_consolidation.creation_trigger` (the business process that created the ticket, e.g. Help Assistant, Contact Form)
- **CHT taxonomy** — NOT on `report_scm_consolidation` (it's an aggregate table; taxonomy is many-to-many per ticket). Needs ticket-level querying:
  - `user_voice__conformed__prod.expose.dim_support_ticket` (ticket grain, has `created_at`, `is_auto_resolved_successful`, `dim_country_dkey`, `dim_plan_dkey`)
  - `LEFT JOIN user_voice__conformed__prod.expose.bridge_support_ticket_cht_category b ON dim_support_ticket_dkey` (resolves the many-to-many category array)
  - `LEFT JOIN user_voice__conformed__prod.expose.dim_cht_category c ON b.dim_cht_category_dkey = c.dim_cht_category_dkey` → use `c.category_level_1` (key_theme), `c.category_level_2` (main_category), `c.category_level_3` (user_issue), `c.category_level_4` (most granular, what Kass's Looker explore calls `cht_category`), `c.moment`
  - **Do not** use `bridge_support_ticket_cht_category.category_level_*` directly — those columns are flagged `TODO: deprecated in favour of dim_cht_category` in the dbt model. Always join through to `dim_cht_category` for the category hierarchy.
  - Filter tickets with `is_auto_resolved_successful = false` to match the "specialist ticket" definition — **validate this is equivalent to `report_scm_consolidation.is_auto_resolved` on a known date/segment before trusting the taxonomy split** (see Caveats).
  - `dim_cht_category_dkey = '-1'` → Unknown category (mirrors the taxonomy convention from the 60-min resolution skill). Keep it in the table, don't lead the narrative with it.
- **Contribution ranking (Kass's method)**: rank taxonomy/channel segments by their **contribution to the CR shift** (rate × mix decomposition, see Step 3 below) — not by raw volume delta. A category can gain volume but still not be "driving" CR if MAU grew proportionally.

## Step 1: Determine the window

- **Weekly**: default to the most recently completed week (Sun-ending, per Kass's convention). Compute via Bash `date` — do not hardcode. `curr_week_end` = most recent Sunday ≤ today; `prev_week_end` = `curr_week_end - 7`; `trend_start` = `curr_week_end - 12 weeks` (12-week trend).
- **Monthly**: default to **MoM: the last fully completed calendar month vs the month before**, unless Kass specifies otherwise. `curr_month_end` = last day of last completed month; `prev_month_end` = last day of the month before that; `trend_start` = `curr_month_end` − 5 months (6-month trend).
- Ask Kass which cadence if the request doesn't say (per CLAUDE.md: ask before writing SQL).

## Step 2: Run the query bundle via data-mcp

Use `execute_query` + `get_query_result`. Templates below use verified columns — substitute dates only. Run the monthly OR weekly bundle depending on cadence.

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
WITH daily AS (
  SELECT calendar_date, SUM(number_of_support_tickets_created) AS tickets
  FROM ds_prod.report.report_scm_consolidation
  WHERE source = 'zendesk' AND NOT is_auto_resolved
    AND calendar_date BETWEEN '{trend_start}' - 6 AND '{curr_week_end}'
  GROUP BY 1
),
wau AS (
  SELECT date_range_upper AS calendar_date, SUM(users_active_rolling_7_days) AS wau
  FROM ds_prod.report.active_user_daily_with_primary_brand_and_mapu_segments
  WHERE is_internal_user = false
    AND date_range_upper BETWEEN '{trend_start}' AND '{curr_week_end}'
  GROUP BY 1
)
SELECT
  w.calendar_date,
  SUM(d.tickets) OVER (ORDER BY w.calendar_date RANGE BETWEEN INTERVAL '6 day' PRECEDING AND CURRENT ROW) AS tickets_rolling_7d,
  w.wau,
  ROUND(100.0 * SUM(d.tickets) OVER (ORDER BY w.calendar_date RANGE BETWEEN INTERVAL '6 day' PRECEDING AND CURRENT ROW) / NULLIF(w.wau, 0), 3) AS contact_rate_pct
FROM wau w
LEFT JOIN daily d ON d.calendar_date = w.calendar_date
WHERE w.calendar_date >= '{trend_start}'
ORDER BY 1
-- Purpose: weekly contact rate headline trend (rolling 7d tickets / WAU)
-- keywords: contact rate, specialist tickets, WAU, report_scm_consolidation
-- CAVEAT: verify the rolling-7d window function against Kass's Looker explore output for one known week before trusting this.
```

### Query B — Channel / entry point drivers (curr + prev period)

```sql
SELECT
  calendar_date,
  channel,
  creation_trigger,
  SUM(number_of_support_tickets_created) AS tickets
FROM ds_prod.report.report_scm_consolidation
WHERE source = 'zendesk' AND NOT is_auto_resolved
  AND calendar_date BETWEEN '{prev_period_start}' AND '{curr_period_end}'
GROUP BY 1,2,3 ORDER BY 1, tickets DESC
-- Purpose: contact rate channel / creation_trigger driver mix
-- keywords: contact rate, channel, creation_trigger, report_scm_consolidation
```

### Query C — CHT taxonomy drivers (curr + prev period)

```sql
SELECT
  DATE_TRUNC('day', t.created_at) AS ticket_date,
  COALESCE(c.category_level_1, 'Unknown') AS key_theme,
  COALESCE(c.category_level_2, 'Unknown') AS main_category,
  COALESCE(c.category_level_4, 'Unknown') AS cht_category,
  COUNT(DISTINCT t.dim_support_ticket_dkey) AS tickets
FROM user_voice__conformed__prod.expose.dim_support_ticket t
LEFT JOIN user_voice__conformed__prod.expose.bridge_support_ticket_cht_category b
  ON t.dim_support_ticket_dkey = b.dim_support_ticket_dkey
LEFT JOIN user_voice__conformed__prod.expose.dim_cht_category c
  ON b.dim_cht_category_dkey = c.dim_cht_category_dkey
WHERE t.is_auto_resolved_successful = false
  AND t.created_at::date BETWEEN '{prev_period_start}' AND '{curr_period_end}'
GROUP BY 1,2,3,4 ORDER BY 1, tickets DESC
-- Purpose: contact rate CHT taxonomy driver decomposition
-- keywords: contact rate, CHT category, dim_cht_category, taxonomy
-- CAVEAT: before trusting this, reconcile total ticket count here against Query B's total for the same period —
-- they should match closely (both claim to be "specialist tickets"). If they diverge by more than a few percent,
-- stop and tell Kass rather than shipping a partial deck (is_auto_resolved_successful may not be a perfect
-- proxy for report_scm_consolidation.is_auto_resolved).
```

### Query D — Free / paid + country segment cuts (appendix, curr + prev period)

Per standing convention: always segment free vs paid for engagement metrics. Run once by `plan_is_free`, once by `country_code` (top 10 by current-period volume).

```sql
SELECT
  calendar_date,
  plan_is_free,
  SUM(specialist_tickets_rolling_30_days) AS specialist_tickets_30d,  -- swap for weekly per Query A (weekly)
  SUM(users_active_rolling_30_days) AS mau,
  ROUND(100.0 * SUM(specialist_tickets_rolling_30_days) / NULLIF(SUM(users_active_rolling_30_days), 0), 3) AS contact_rate_pct
FROM ds_prod.report.report_uhe_consolidation
WHERE calendar_date IN ('{prev_period_end}', '{curr_period_end}')
GROUP BY 1,2 ORDER BY 1, specialist_tickets_30d DESC
-- Purpose: contact rate free/paid segment cut
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

- **Weekly WAU source is not yet independently verified** — `active_user_daily_with_primary_brand_and_mapu_segments` is the standard Canva-wide WAU table and matches the LookML measure's shape, but has not been checked row-for-row against Kass's Looker explore output. On first run, pull one known week and compare against the Looker number before trusting the trend.
- **CHT taxonomy ticket filter is not yet independently verified** — `dim_support_ticket.is_auto_resolved_successful = false` is used as the "specialist ticket" filter for taxonomy decomposition, but `report_scm_consolidation.is_auto_resolved` (used for the headline and channel cuts) comes from a different, pre-aggregated model. Reconcile total counts between Query B and Query C for the same period before trusting the taxonomy split (see Query C's inline caveat).
- Contact rate as defined here is **already Specops-only** (excludes auto-resolved/automation tickets by construction). Don't re-apply an Automation vs Specops filter on top of it — it's already applied. If a broader "all tickets regardless of resolution path" view is wanted, say so explicitly and use the unfiltered ticket counts instead, labelled as a different metric.
- Always segment free vs paid for CR/engagement comparisons per standing convention — don't skip Query D even if not asked, unless Kass says otherwise.
- Report volumes as absolute + relative always: "1,200 tickets (+8%)", not just one or the other.
- `dim_cht_category_dkey = '-1'` → 'Unknown' category. Keep in the table, don't lead the narrative with it.
- If any query returns null/zero unexpectedly, stop and tell Kass rather than shipping a partial deck.
