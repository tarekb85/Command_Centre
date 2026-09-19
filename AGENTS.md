# AGENTS.md — Business & Investment Command Centre

This repo is the source for a single-page dashboard (`index.html`), deployed
via GitHub Pages, backed entirely by Supabase (Postgres + PostgREST + Auth).
There is no build step and no server-side code — everything is client-side
JS talking directly to Supabase's REST API.

## Source of Truth Hierarchy

When any of these disagree, resolve in this order:

1. **Live Supabase schema/data** — authoritative for what currently exists
   and the current data. Query it directly; never assume.
2. **This file + `docs/INVESTMENT_SYSTEM.md`** — authoritative for intended
   architecture, operating rules, and design decisions. Not authoritative
   for current data state, and not authoritative for what the dashboard
   actually renders.
3. **Dashboard code (`index.html`)** — authoritative for what the current
   UI actually implements. The dashboard can lag behind both Supabase and
   the documentation — check the code itself, don't assume from the docs.
4. **Chat history / AI memory** — contextual only, never authoritative over
   any of the above. A prior conversation's summary of the system is not a
   substitute for checking 1–3 directly.

This ordering mattered concretely on 2026-09-18: Supabase had `project_
analysis`, `potential_swaps`, and `reminders` built with real data while
the dashboard fetched none of them — a live example of Supabase being
ahead of the code. Phase 4 (2026-09-18) closed that specific gap by adding
view-only Research/Opportunities/Catalysts tabs. The general risk this
illustrates still applies going forward: **check what the dashboard code
actually fetches, don't assume from this file or from `docs/
INVESTMENT_SYSTEM.md`** — the next gap of this kind won't announce itself
either. See `docs/INVESTMENT_SYSTEM.md` for current per-table wiring
status.

## Before you change anything

1. **Inspect the live Supabase schema before touching it.** Do not trust
   this file, `docs/INVESTMENT_SYSTEM.md`, or your own prior turns in a
   chat to reflect current reality. Schema has drifted from documentation
   before (see Known Gotchas). Query `information_schema.tables` /
   `information_schema.columns` directly.
2. **Read `docs/INVESTMENT_SYSTEM.md`** for the full data model, the
   Analyse → Record → Conviction → Potential Swap → Execute → Review
   lifecycle this system is built around, and which parts of that lifecycle
   are actually wired into the dashboard vs. schema-only.
3. **Never assume the dashboard UI reflects the full schema — check the
   code, every time.** As of 2026-09-18, `project_analysis`,
   `potential_swaps`, and `reminders` are fetched (view-only) by the
   Research/Opportunities/Catalysts tabs — but none of the three have
   create/edit/delete UI yet; recording an analysis or resolving a
   reminder still requires direct SQL. A table's existence, or even its
   presence in a fetch call, doesn't tell you what operations the UI
   actually supports. Check `docs/INVESTMENT_SYSTEM.md`'s Current Known
   State section for the latest per-table wiring status, and re-verify
   against the live code if it matters.

## Triggering a project analysis

When asked to "run analysis on X" or "analyze X and log it" (with or
without mentioning debate), no further explanation of the process should
be needed — follow this sequence. It's proven, not theoretical: this is
exactly what ran successfully for AURORA on 2026-09-19.

1. Read this file and `docs/INVESTMENT_SYSTEM.md` first.
2. Query live `project_analysis` for existing rows on that token before
   writing anything. Never overwrite an existing row — a genuine
   re-analysis is a new row linked via `supersedes_analysis_id`.
3. Conduct independent research and form a real, evidenced view — cite
   specifics (dates, figures, on-chain data), not vague direction. Never
   fabricate a score, catalyst, or risk that wasn't actually found.
4. Write one new `project_analysis` row: `analysis_by` = the specific AI
   name (e.g. `'Claude'`, not `'AI'`), `analysis_type = 'FULL'`,
   `framework_version = 'v1'`, narrative fields populated genuinely from
   what the research actually found. **Sub-scores, `overall_rating`, and
   `next_review_date` are not mandatory fields to fill on every analysis
   — populate them only when the analysis genuinely supports a specific
   number or date.** Forcing a score or a review date just to satisfy a
   template creates a data-quality problem worse than leaving it null;
   `null` honestly means "not yet assessed," a fabricated value doesn't.
5. If a debate was explicitly requested ("...and log it for debate"):
   check whether an *unresolved* `debate_id` already exists for this
   token (a debate with contribution rows but no `DEBATE_SYNTHESIS` row
   yet). If one exists, reuse it — never fragment one debate across
   multiple ids. Otherwise generate a fresh `debate_id` and set it on
   this row.
6. Report back: the row id, whether an existing analysis was found, and
   — if this was one side of a debate — that the state is now correctly
   shared and no copy/paste is needed for the other AI to find it.

Never average scores across independent analyses, never force consensus
between them, and never assume a plain "analyze X" implies a debate
unless the person actually asked for one.

**Debate round-by-round mechanics are intentionally not documented here
yet** — challenge/response labeling, round caps, and stopping rules are
still being proven in practice (AURORA is the first live test) before
being locked in. Document those separately once something real has
actually worked.

## Deployment

- Repo: `tarekb85/Command_Centre` (public)
- Live: `https://tarekb85.github.io/Command_Centre/`
- Deploy = push `index.html` to `main` via the GitHub Contents API. No CI,
  no build. GitHub Pages rebuilds automatically, usually within ~30s.
- Deploy token: fine-grained PAT, scoped to this repo only,
  `Contents: Read and write`. Do not request or use a broader-scoped token
  for this repo.
- Validate JS syntax locally (`node -e "new Function(scriptContent)"`)
  before every push — there's no CI to catch a broken commit.

## Database

- Supabase project: `gdrhkvrjarnhrhhoxxum` ("Business & Investment"),
  region eu-west-1.
- Auth: single user account, RLS on every table scoped to that specific
  `auth.uid()` — never write a policy scoped to "any authenticated user."
  Public sign-ups are disabled on this project; do not re-enable them.
- The anon/publishable key is intentionally visible in the client-side
  code — this is normal for a client-side Supabase app and is safe *only
  because* RLS is locked to one user. Do not "fix" this by hiding the key;
  fix it by keeping RLS tight if it's ever loosened.

## Known gotchas (verified, not hypothetical)

- A sibling project (Life OS, different Supabase project) had a schema
  migration silently break its deployed dashboard because two tables were
  merged into one without the deployed code being updated. The same class
  of risk applies here: a schema change in Supabase does not automatically
  propagate to this repo's `index.html`. Always cross-check after any
  schema change that the dashboard's fetch queries still match.
- GitHub Pages build status can be checked via
  `GET /repos/tarekb85/Command_Centre/pages/builds/latest` — use this to
  confirm a push actually deployed rather than assuming.

## What not to do

- Don't create a new table if an existing one already serves the purpose —
  check `docs/INVESTMENT_SYSTEM.md` and the live schema first.
- Don't overwrite a `project_analysis` row to "update" a thesis — see the
  versioning model in `docs/INVESTMENT_SYSTEM.md` (`supersedes_analysis_id`).
  Historical analysis is preserved, not overwritten.
- Don't widen RLS or re-enable sign-ups on this Supabase project.
- Don't invent schema details not confirmed live. If uncertain, query the
  schema — don't guess from this file or from chat history.
