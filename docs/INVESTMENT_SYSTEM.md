# Business & Investment System — Architecture & Operating Model

**What this document is:** the canonical description of *intended*
architecture, operating rules, and design decisions for this system. It is
**not** the source of truth for current data — that's live Supabase. It is
**not** a description of what the deployed dashboard actually shows —
that's the dashboard's own code. See `AGENTS.md`'s "Source of Truth
Hierarchy" for the full ordering; this file occupies position 2 of 4, not
position 1.

Sections below are labeled CONFIRMED (verified live against Supabase on
2026-09-18), INTENDED (designed but not yet built, or not yet wired into
the dashboard), or UNVERIFIED (could not be checked from available
tooling). Do not treat INTENDED as if it were CONFIRMED, and do not treat
anything in this file as an assertion about current data — for that,
query Supabase directly.

## Philosophy

This is designed as a **long-term investment memory**, not a live ranking
board. The system exists to preserve *why* a decision was made, not just
the current rating — so a conviction from six months ago can be revisited
with its original reasoning intact, not just a number that's since changed.

The intended lifecycle:

**Analyse → Record → Conviction → Potential Swap → Execute → Review**

| Stage | Purpose | Primary table (CONFIRMED to exist) |
|---|---|---|
| Analyse | Deep-dive research on a token/project | `project_analysis` |
| Record | Snapshot the analysis with attribution & scoring | `project_analysis` |
| Conviction | Current dashboard-facing view/ranking | `conviction_items` |
| Potential Swap | A candidate rotation, not yet executed | `potential_swaps` |
| Execute | The rotation actually happened | `rotations` + `rotation_basket_items` |
| Review | Scheduled re-checks, unlocks, catalysts | `reminders` |

## Current Known State (CONFIRMED — verified live 2026-09-18)

This section describes what was true in Supabase at verification time. For
current data, re-query Supabase — do not treat the numbers below as live.

### Tables actively fetched and rendered by the deployed dashboard

| Table | Rows at verification | Notes |
|---|---|---|
| `conviction_items` | 36 | Generalist/TradFi/Watchlist lists |
| `portfolio_holdings` | 11 | Actual current holdings |
| `rotations` | 4 | Executed rotations (FLR, CFG, XPL→CFG, ONDO→CFG) |
| `rotation_basket_items` | 11 | Line items within each rotation's basket |
| `calendar_events` | 12 | General calendar/event tracking |
| `open_items` | 14 | Ad-hoc backlog |
| `multi_ai_rounds` | 1 | Metadata for a multi-AI convergence round |
| `multi_ai_rankings` | 40 | Per-AI per-token rankings for a round |

`project_context` (1 row) also exists — a free-text running-notes field,
not part of the dashboard UI, used for chat-session continuity.

### Tables that exist in Supabase but are NOT wired into the dashboard yet

This is the most important gap in the system right now. Per
`AGENTS.md`'s Source of Truth Hierarchy, Supabase being ahead of the
dashboard here is a fact about current state (Supabase, position 1) that
this document (position 2) is reporting on — not something this document
resolves. **Documenting this gap is in scope; building the dashboard UI
for it is not.**

| Table | Rows at verification | Gap |
|---|---|---|
| `project_analysis` | 0 | Full schema built (see below). Zero rows recorded. No dashboard tab reads or writes it. |
| `potential_swaps` | 5 | Schema built. 5 rows exist (all inserted 2026-09-18, all `status='WATCH'`, all `source_analysis_id`/`target_analysis_id` NULL — no linked analysis yet). No dashboard tab reads or writes it. |
| `reminders` | 1 | Investment-specific reminders (unlocks/governance/catalysts) — distinct from `calendar_events`. 1 row (`MON` unlock, 2026-11-24, severity high). No dashboard tab reads or writes it. |

**Practical implication:** if you're asked to "check the reminders" or
"look at potential swaps," the data lives in Supabase and is queryable via
SQL, but nothing on the live dashboard currently shows it. Don't assume a
user has seen this data just because it's "in the system."

### Confirmed live relationships (foreign keys)

- `conviction_items.latest_analysis_id` → `project_analysis.id`
- `potential_swaps.source_analysis_id` → `project_analysis.id`
- `potential_swaps.target_analysis_id` → `project_analysis.id`
- `project_analysis.supersedes_analysis_id` → `project_analysis.id` (self-referential — the versioning chain)

All four exist as real FK constraints in the live schema, but since
`project_analysis` has 0 rows, none are currently populated with a value.

### RLS pattern (CONFIRMED)

Every table above has RLS enabled with a policy scoped to one specific
`auth.uid()` (not "any authenticated user," not open to anon). The three
newer tables (`project_analysis`, `potential_swaps`, `reminders`) use
`(SELECT auth.uid()) = '<uuid>'` — a scalar-subquery form that's a minor
performance optimization over the plain `auth.uid() = '<uuid>'` form used
on the earlier tables. Both are equivalent in effect; the newer form is
preferable for any new table going forward.

## Table Reference

### `project_analysis` (CONFIRMED schema, 0 rows)

The core research record. One row per analysis snapshot — not per token,
since a token can and should accumulate multiple analyses over time as
the thesis evolves (see Versioning below).

Fields confirmed live:
- `token`, `project_name`, `category`, `cgid`
- `analysis_date`, `analysis_by` (free text — e.g. "Tarek + Claude",
  "Multi-AI", "Tarek + ChatGPT"), `analysis_type` (default `'FULL'`),
  `framework_version` (default `'v1'`)
- Narrative fields: `thesis`, `bull_case`, `bear_case`, `key_catalysts`,
  `key_risks`, `token_value_capture`, `adoption_traction`,
  `competitive_moat`, `liquidity_notes`, `tokenomics_notes`,
  `valuation_notes`, `conclusion`, `snapshot_notes`
- Scoring (all numeric 0–10, all nullable):
  `score_asymmetry`, `score_probability_weighted_return`,
  `score_token_value_capture`, `score_supply_unlocks`,
  `score_catalysts`, `score_adoption_traction`,
  `score_competitive_moat`, `score_liquidity`, `score_downside_risk`
- `overall_rating` (numeric 0–10), `conviction` (LOW/MEDIUM/HIGH),
  `status` (WATCH/HOLD/ACCUMULATE/EXIT)
- Market snapshot at time of analysis: `price_usd`, `market_cap_usd`,
  `fdv_usd`, `tvl_usd`, `circulating_supply`, `max_supply`
- `source_urls` (text array), `tags` (text array)
- `next_review_date`
- `supersedes_analysis_id` — see Versioning below

**INTENDED, not yet enforced by any constraint or code:** analyses should
never be overwritten to reflect a re-evaluation. A new re-evaluation is a
*new row*, linked via `supersedes_analysis_id` back to the one it replaces.
This is a process rule, not something the database currently prevents —
nothing stops an UPDATE from silently destroying history today. If this
matters enough to enforce mechanically, that would need a trigger or an
application-level convention; neither exists yet (UNVERIFIED whether this
has been discussed elsewhere).

### `conviction_items` (CONFIRMED, 36 rows, live in dashboard)

The current-state dashboard view — Generalist/TradFi/Watchlist lists.
`latest_analysis_id` FK exists but is not currently populated for any row
(consistent with `project_analysis` being empty). Once analyses start
being recorded, this is the intended link from "current conviction" back
to "the research that justifies it."

### `potential_swaps` (CONFIRMED, 5 rows, NOT live in dashboard)

A candidate rotation that hasn't been executed. Fields: `source_token`,
`target_token`, `status` (IDEA/WATCH/READY/EXECUTED/ABANDONED), `priority`
(1–5), `conviction` (0–10), `source_sell_pct`, `proceeds_share_pct`,
`target_weight_pct`, `thesis`, `why_now`, `trigger`, `invalidation`,
`catalysts`, `risk_notes`, `review_date`, `last_reviewed_at`, plus the two
`*_analysis_id` links to `project_analysis`.

Current 5 rows (at verification) are all `ADA → {CC, ZRO, NEAR, CELO, AKT}`,
status `WATCH`, priorities 1–5, inserted as a batch on 2026-09-18. No
conviction score or linked analysis set yet on any of them.

**UNVERIFIED:** target_token `CC` on the first row — could not confirm
whether this is a real ticker or a typo/placeholder. Don't assume either
way; check with Tarek or search before treating it as a real asset.

### `rotations` + `rotation_basket_items` (CONFIRMED, live in dashboard)

The executed side of a swap — this is the "Execute" stage. `rotations` (4
rows: FLR, CFG, XPL→CFG, ONDO→CFG) holds what was sold; each row's basket
of what was bought lives in `rotation_basket_items` (11 rows). Fully live
in the dashboard's Rotations tab, including a buyback-comparison feature.

### `reminders` (CONFIRMED, 1 row, NOT live in dashboard)

Investment-specific catalysts/deadlines — distinct from the general
`calendar_events` table. `reminder_type` enum: UNLOCK, GOVERNANCE,
CATALYST, REVIEW, DEADLINE, OTHER. `severity`: high/medium/low. `status`:
OPEN/DONE/DISMISSED/SNOOZED. Supports `recurrence_rule` and `snooze_until`
for recurring or deferred reminders. Current single row (at verification):
a MON token unlock flagged high-severity for 2026-11-24.

### `portfolio_holdings`, `calendar_events`, `open_items`, `multi_ai_rounds`, `multi_ai_rankings`, `project_context`

All CONFIRMED live and actively used by the dashboard. Schemas are
straightforward — query live `information_schema` for exact columns rather
than relying on this document, which doesn't reproduce them here since
they don't carry the same new-vs-intended ambiguity as the tables above.

## Operating Rules (INTENDED — process rules, not database-enforced)

These are stated conventions for how this system should be worked with.
None of them are enforced by a database constraint today; they rely on
whoever (human or agent) is operating on the system following them.

1. Supabase is authoritative for current state — not this document, not a
   chat's memory of a prior session.
2. This documentation is canonical for intended architecture and process —
   not for what currently exists. The live schema is authoritative for
   that.
3. Never assume live schema matches this document. Inspect before
   changing anything.
4. Never create a duplicate table when an existing one already serves the
   purpose.
5. Never overwrite a meaningful historical `project_analysis` row on
   re-evaluation — create a new row and link it via `supersedes_analysis_id`.
6. Preserve `analysis_by` attribution on every analysis.
7. Preserve the *reasoning* behind a decision, not just the final rating —
   this is the entire point of `project_analysis`'s narrative fields.
8. Distinguish factual/current external data (price, TVL, supply) from
   analysis/opinion (thesis, scores, conviction).
9. For time-sensitive crypto data, verify current figures before recording
   them as fact — don't record stale or assumed numbers into
   `project_analysis`.
10. Any schema change should be deliberate, documented here afterward, and
    verified against the live database after implementation.
11. Avoid over-engineering — don't add structure the system doesn't need
    yet on the basis of "future-proofing."
12. Keep the system extensible for future agents, additional analysis
    frameworks, and additional portfolio workflows.

## What could not be verified

- Who built `project_analysis`, `potential_swaps`, and `reminders`, and
  when the design was decided — confirmed to exist and when the data was
  inserted (2026-09-18), not the session where the schema was designed.
- `potential_swaps` row 1's target_token `CC` — flagged as unverified
  rather than guessed at.
- Whether `supersedes_analysis_id` non-overwrite behavior is meant to be
  enforced by a trigger eventually, or stays a human/agent convention
  indefinitely.
- Whether the dashboard is intended to eventually grow tabs for
  Analysis/Potential Swaps/Reminders, or whether those are meant to stay
  Supabase/SQL-only workflows. Described as a current gap, not a plan.
- `framework_version`'s intended versioning path — the column exists with
  a default of `'v1'`, but no v2 or migration logic exists anywhere
  checkable.
