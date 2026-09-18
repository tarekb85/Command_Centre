# Business & Investment System — Architecture & Operating Model

**What this document is:** the canonical description of *intended*
architecture, operating rules, and design decisions for this system. It is
**not** the source of truth for current data — that's live Supabase. It is
**not** a description of what the deployed dashboard actually shows —
that's the dashboard's own code. See `AGENTS.md`'s "Source of Truth
Hierarchy" for the full ordering; this file occupies position 2 of 4, not
position 1.

Sections below are labeled CONFIRMED (verified live against Supabase on
2026-09-18, re-verified same day after Phase 4 dashboard work), INTENDED
(designed but not yet built, or not yet wired into the dashboard), or
UNVERIFIED (could not be checked from available tooling). Do not treat
INTENDED as if it were CONFIRMED, and do not treat anything in this file
as an assertion about current data — for that, query Supabase directly.

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

Three more tables were added to the dashboard in Phase 4 — see below.

### Tables now wired into the dashboard as of Phase 4 (2026-09-18)

Updated from the original audit: `project_analysis`, `potential_swaps`,
and `reminders` are now read (view-only) by the dashboard, in three new
tabs — Research, Opportunities, and Catalysts — plus a compact Overview
tab. This closes the gap the original Phase 1 audit flagged. Phase 4 did
not add any write/edit/delete functionality for these three tables; that
remains a possible future phase, not yet built.

| Table | Rows at Phase-4 verification | Notes |
|---|---|---|
| `project_analysis` | 20 | All 20 are `analysis_type = 'MIGRATED_CONVICTION_NOTE'` — preserved research notes migrated from the old conviction layer, not newly researched formal analyses. None have `overall_rating`, `conviction`, `bull_case`, or `bear_case` populated; all carry a `thesis` field. `analysis_by` on these reads "Historical dashboard note — attribution not fully verified" rather than a specific person/AI. |
| `potential_swaps` | 5 | The ADA trim basket: 30% source sell on every row, `proceeds_share_pct` split 30/25/20/15/10 across CC/ZRO/NEAR/CELO/AKT (retaining 70% ADA), all `status='WATCH'`. `target_analysis_id` is populated for 4 of 5 (not CC, which has no resolvable project). **`source_analysis_id` is populated on none of the 5** — there is currently no analysis backing the ADA side of the trim. Narrative fields (thesis/why_now/trigger/invalidation/catalysts/risk_notes) are null on all 5 rows. |
| `reminders` | 2 | ZRO cliff unlock (2026-09-19) and MON cliff unlock (2026-11-24), both `severity='high'`, both `status='OPEN'`, both fully populated including `action`, `details`, and `source_url`. |

**"CC" remains intentionally unresolved** — the dashboard renders it as-is with a small "(ticker unresolved)" label. Nothing in this system should guess what it stands for.

### Confirmed live relationships (foreign keys)

- `conviction_items.latest_analysis_id` → `project_analysis.id`
- `potential_swaps.source_analysis_id` → `project_analysis.id`
- `potential_swaps.target_analysis_id` → `project_analysis.id`
- `project_analysis.supersedes_analysis_id` → `project_analysis.id` (self-referential — the versioning chain)

All four exist as real FK constraints in the live schema. Population as of
Phase-4 verification: `conviction_items.latest_analysis_id` — 30 of 36
populated. `potential_swaps.target_analysis_id` — 4 of 5 populated.
`potential_swaps.source_analysis_id` — 0 of 5 populated (no analysis
currently backs the ADA side of the trim basket).
`project_analysis.supersedes_analysis_id` — 0 of 20 populated (every
current row is first-generation; no analysis has yet been superseded by
a later one).

### RLS pattern (CONFIRMED)

Every table above has RLS enabled with a policy scoped to one specific
`auth.uid()` (not "any authenticated user," not open to anon). The three
newer tables (`project_analysis`, `potential_swaps`, `reminders`) use
`(SELECT auth.uid()) = '<uuid>'` — a scalar-subquery form that's a minor
performance optimization over the plain `auth.uid() = '<uuid>'` form used
on the earlier tables. Both are equivalent in effect; the newer form is
preferable for any new table going forward.

## Table Reference

### `project_analysis` (CONFIRMED, 20 rows, live in dashboard's Research tab as of Phase 4)

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
- `debate_id` (uuid, nullable) — added 2026-09-18. Groups multiple
  `project_analysis` rows that belong to the same opt-in formal debate
  (e.g. a ChatGPT row and a Claude row on the same token, plus a
  synthesis row). NULL for the overwhelming majority of rows — debates
  are deliberately rare, not the default path for recording analysis.
  No index yet; not justified at current volume (see migration note
  below). Not a foreign key to anything — it's a bare grouping value,
  not a reference to a debate-sessions table, since none exists or is
  currently planned.

**INTENDED, not yet enforced by any constraint or code:** analyses should
never be overwritten to reflect a re-evaluation. A new re-evaluation is a
*new row*, linked via `supersedes_analysis_id` back to the one it replaces.
This is a process rule, not something the database currently prevents —
nothing stops an UPDATE from silently destroying history today. Confirmed:
all 20 current rows have `supersedes_analysis_id = null` — no analysis has
been superseded yet, so this chain is untested in practice, not just
unenforced. If this matters enough to enforce mechanically, that would
need a trigger or an application-level convention; neither exists yet
(UNVERIFIED whether this has been discussed elsewhere).

**Debate / multi-AI disagreement convention (INTENDED — a usage
convention, not a schema constraint):** Formal debates are opt-in, not
the default way analysis gets recorded. When Tarek explicitly requests
one (e.g. "ChatGPT and Claude, debate MON"), each AI's independent take
is its own `project_analysis` row, same `debate_id`, `analysis_by`
identifying that AI. A third row — the synthesis — carries the resolved
view and should be written with an `analysis_by` value that says what it
is, e.g. `"Synthesis — ChatGPT + Claude debate, resolved by Tarek"`, not
left ambiguous with a single AI's name. The synthesis row reuses existing
narrative fields rather than needing new ones: `thesis` holds the
resolved view, `key_risks`/`bear_case` hold the actual points of
divergence between the two AIs (not a generic risk list), and
`snapshot_notes` holds any open/unresolved question. Genuine disagreement
between AIs is preserved as-is in their separate rows — it is never
averaged into a blended score, and consensus is never forced. Tarek's own
decision lives in `conviction_items`, linked via `latest_analysis_id` to
the synthesis row, not directly to either AI's individual row — that's
what keeps the chain traceable: conviction → synthesis → both original
independent analyses.

**`multi_ai_rounds` and `multi_ai_rankings` are not used for debates, and
this is deliberate, not an oversight.** Those two tables represent a
different, pre-existing concept — a portfolio-wide convergence exercise
where a fixed set of AIs (`multi_ai_rankings.ai_name` is CHECK-constrained
to `claude`/`grok`/`chatgpt`/`gemini`, confirmed live) rank the same list
of tokens in one shared exercise. A debate about one project is a
different shape of question — asynchronous, not necessarily the same
prompt to each side, and including a synthesis step those tables were
never built to hold. Repurposing them for debates would make one table
responsible for two different concepts, which this system has otherwise
deliberately avoided. Do not extend `multi_ai_rankings`' `ai_name` CHECK
constraint to accommodate a "Tarek" or "synthesis" row for this purpose —
that content belongs in `project_analysis` instead.

### `conviction_items` (CONFIRMED, 36 rows, live in dashboard)

The current-state dashboard view — Generalist/TradFi/Watchlist lists.
`latest_analysis_id` is populated on 30 of 36 rows as of Phase 4. As of
Phase 4, a card whose token has a linked analysis shows a small "🔬
analysis" link that jumps to that analysis in the Research tab — the one
piece of existing-card UI touched by Phase 4, purely additive (see
`AGENTS.md`-referenced delivery notes for this phase).

### `potential_swaps` (CONFIRMED, 5 rows, live in dashboard's Opportunities tab as of Phase 4)

A candidate rotation that hasn't been executed. Fields: `source_token`,
`target_token`, `status` (IDEA/WATCH/READY/EXECUTED/ABANDONED), `priority`
(1–5), `conviction` (0–10), `source_sell_pct`, `proceeds_share_pct`,
`target_weight_pct`, `thesis`, `why_now`, `trigger`, `invalidation`,
`catalysts`, `risk_notes`, `review_date`, `last_reviewed_at`, plus the two
`*_analysis_id` links to `project_analysis`.

Current 5 rows: the ADA trim basket — `ADA → CC` (priority 1, 30% proceeds
share), `ADA → ZRO` (priority 2, 25%), `ADA → NEAR` (priority 3, 20%),
`ADA → CELO` (priority 4, 15%), `ADA → AKT` (priority 5, 10%) — proceeds
shares sum to 100%. `source_sell_pct` is 30% flat on every row (retaining
70% ADA). All `status='WATCH'`. `target_analysis_id` is populated on 4 of
5 (ZRO, NEAR, CELO, AKT — not CC, which has no resolvable project).
`source_analysis_id` is null on all 5 — no analysis currently backs the
ADA side. `conviction` and all narrative fields (thesis/why_now/trigger/
invalidation/catalysts/risk_notes) are null on all 5 rows.

**UNVERIFIED:** target_token `CC` — could not confirm whether this is a
real ticker or a typo/placeholder. The dashboard renders it as-is with an
"(unresolved ticker)" label rather than guessing. Don't assume either way;
check with Tarek or search before treating it as a real asset.

### `rotations` + `rotation_basket_items` (CONFIRMED, live in dashboard)

The executed side of a swap — this is the "Execute" stage. `rotations` (4
rows: FLR, CFG, XPL→CFG, ONDO→CFG) holds what was sold; each row's basket
of what was bought lives in `rotation_basket_items` (11 rows). Fully live
in the dashboard's Rotations tab, including a buyback-comparison feature.

### `reminders` (CONFIRMED, 2 rows, live in dashboard's Catalysts tab as of Phase 4)

Investment-specific catalysts/deadlines — distinct from the general
`calendar_events` table. `reminder_type` enum: UNLOCK, GOVERNANCE,
CATALYST, REVIEW, DEADLINE, OTHER. `severity`: high/medium/low. `status`:
OPEN/DONE/DISMISSED/SNOOZED. Supports `recurrence_rule` and `snooze_until`
for recurring or deferred reminders. Current 2 rows, both `severity='high'`
and `status='OPEN'`: a ZRO cliff unlock (event 2026-09-19) and a MON cliff
unlock (event 2026-11-24), both with fully populated `action`, `details`,
and `source_url` fields.

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
  indefinitely. Confirmed only that it's currently unused (0 of 20 rows).
- `framework_version`'s intended versioning path — the column exists with
  a default of `'v1'`, but no v2 or migration logic exists anywhere
  checkable.

## Known gap, not unverified — stated plainly

Phase 4 (2026-09-18) added view-only dashboard access to `project_analysis`,
`potential_swaps`, and `reminders`. It did **not** add any create/edit/
delete UI for these three tables — recording a new analysis, adding a
swap candidate, or resolving a reminder still requires direct SQL access
(e.g., via an agent with a Supabase connection) rather than the dashboard
itself. Whether that remains permanent or becomes a future phase is an
open product decision, not something this document resolves.
