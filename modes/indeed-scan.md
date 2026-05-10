# Mode: indeed-scan -- Indeed Job Search via claude.ai Connector

## Purpose

Use the **`claude.ai Indeed`** MCP connector to systematically discover entry-level / cap-exempt-friendly Indeed listings matching the candidate's archetype, deduped against `data/applications.md`. Fills the gap left by `/career-ops scan`'s WebSearch-only Indeed coverage (which only sees Google-indexed jobs and lacks Indeed's native filters).

This mode is the **discovery** half. The **apply** half is unchanged — for any viable Indeed URL, run `/career-ops {url}` for auto-pipeline evaluation, then `/career-ops apply` to fill the form.

## Inputs

- `portals.yml` — `title_filter.positive` and `negative` lists (drives query construction).
- `data/applications.md` — dedupe target (don't propose URLs already in tracker).
- `config/profile.yml` — candidate location preferences, OPT context, archetype hints.
- **Indeed MCP tools** — must be enabled in the project's claude.ai connectors. Look for tools matching `mcp__claude_ai_Indeed__*` (typically: Job Search, Job Details, Company Information, Get Resume).

## Step 0 — Verify Indeed MCP availability

Before doing anything, confirm Indeed tools are loaded in the current session. Look for tool names matching `mcp__claude_ai_Indeed__*` (the exact name depends on the connector configuration — could be `INDEED_JOB_SEARCH`, `JobSearch`, `job_search`, etc.).

If Indeed MCP is NOT available:

> "Indeed connector is not enabled for this Claude Code session. To enable it:
> 1. Visit https://claude.ai/customize/connectors
> 2. Click `Indeed` (already OAuth-completed account-level)
> 3. Confirm the toggle for THIS project is on (might require restart of the Claude Code session)
> 4. Re-run `/career-ops indeed-scan`."

Exit gracefully. Do NOT attempt to proceed.

## Step 1 — Build the search batch from `portals.yml`

Read `portals.yml` and group the `title_filter.positive` keywords into 6-8 logical queries (NOT all 50+ keywords as one query, NOT each keyword as a separate query). Suggested groupings:

| Group | Keywords (OR-joined) |
|---|---|
| Analyst (entry) | "business analyst", "operations analyst", "innovation analyst", "data analyst", "strategy analyst" |
| AI/Tech (entry) | "AI consultant", "AI strategy", "AI implementation", "AI program manager", "junior LLM engineer", "AI fellow" |
| Coordinator (cap-exempt) | "program coordinator", "project coordinator", "research coordinator", "administrative coordinator" |
| Government / public | "government operations", "government analyst", "purchasing analyst", "vendor services analyst" |
| Sales/BD (entry) | "business development representative", "BDR", "sales development representative", "SDR" |
| Bilingual (differentiator) | "bilingual spanish", "latin america", "LATAM" entry |
| Fellowship | "fellowship", "innovation fellow", "AI for good" |
| Nonprofit | "nonprofit operations", "nonprofit innovation", "social impact analyst" |

For each group, parameters to pass to Indeed Job Search:

```yaml
keywords: "{joined-or-list}"
location: "United States"
remote: true   # also surface remote
posted: "last_14_days"   # don't return stale postings
job_type: "fulltime"    # also "internship" for the Internship-relevant groups
experience: "entry_level"
limit: 25
```

For **internship**-relevant groups (Coordinator, AI/Tech, Bilingual, Government), run a SECOND query with `job_type: "internship"` to capture both. Internships pass visa filters more often than FT (see Patterns log in `applications.md`: SoF FT auto-DQ, internships accepted).

## Step 2 — Dispatch Indeed Job Search batch

For each group + job_type combination (≤16 total queries), call the Indeed Job Search MCP tool. **Run in parallel where possible** to minimize latency.

Capture for each result: `job_id`, `title`, `company`, `location`, `posted_date`, `salary_range` (if exposed), `job_url`, `description_snippet`.

## Step 3 — Apply filter pipeline

For each raw result:

1. **Negative filter:** drop if title contains any of `portals.yml::title_filter.negative` (Senior, Staff, Principal, Lead, Director, VP, TPM, etc.).
2. **Citation filter:** drop if title contains "must be a US citizen", "US citizens only", "security clearance", etc.
3. **Dedupe vs tracker:** drop if `job_url` or `(company, role)` pair matches an existing row in `data/applications.md`.
4. **Posted-age filter:** drop anything posted >30 days ago (low live-rate).
5. **Salary sanity:** if salary is exposed AND <$40K (cost-of-living floor) AND not internship, flag as low-priority but don't drop.

## Step 4 — Enrich top candidates with Job Details

For the **top 30** post-filter (sorted by recency × archetype-match score), call Indeed Job Details to fetch full JD. **Cap at 30** to control token spend.

The full JD enables better archetype-fit scoring in Step 5.

## Step 5 — Rank by composite score

Composite score per result, weighted:

| Weight | Signal | Source |
|---|---|---|
| 0.30 | Archetype-fit (matches one of `target_roles.archetypes` in profile.yml) | JD title + description |
| 0.20 | Cap-exempt employer signal (university / government / 501(c)(3) / nonprofit) | JD body + company info |
| 0.15 | H1B sponsorship explicit mention | JD body |
| 0.15 | Entry-level explicit mention | JD title + body |
| 0.10 | Bilingual EN/ES requirement | JD body |
| 0.10 | Salary transparency + reasonable range | JD salary field |

Score 0-1 normalized.

## Step 6 — Present ranked results

Show top 20 in a compact table:

```
Indeed scan — {date} (window: posted last 14 days)

{N total results before filters} → {N after filters} → showing top 20

| # | Score | Company                  | Role                                   | Loc/Remote   | $       | Indicators           |
| 1 | 0.82  | AI Now Institute         | Communications Associate (NYC)         | NYC          | $80-100k| 501(c)(3), bilingual |
| 2 | 0.78  | State of Florida (DOT)   | Govt Operations Consultant Internship  | Tallahassee  | $20/hr  | cap-exempt internship|
...

(plus 6 hidden 0.4-0.6 score for stretch consideration if you want them: type `show all`)
```

After the table, prompt:

> "What now?
> - `add top N to pipeline` (e.g. `add top 10 to pipeline`) → appends to data/pipeline.md for `/career-ops pipeline` evaluation later
> - `evaluate #1, #3, #5` → runs `/career-ops {url}` (auto-pipeline) on those specific entries now
> - `show all` → reveals 0.4-0.6 score stretch entries
> - `refine query: <feedback>` → adjusts groups + reruns (e.g., 'fewer SDR roles, more analyst')
> - `done` → exit"

## Step 7 — Persist scan history (optional)

If `data/scan-history.tsv` exists (career-ops standard), append a row:

```
date<TAB>scan_method<TAB>queries_run<TAB>raw_results<TAB>after_filter<TAB>added_to_pipeline
2026-05-10<TAB>indeed_connector<TAB>14<TAB>312<TAB>47<TAB>10
```

This feeds the patterns analysis (`/career-ops patterns`) over time.

## Rules

- **Conservative dedupe.** Better to skip an arguable duplicate than to clutter `pipeline.md`. Match on URL primarily, fall back to `(company, role)` fuzzy match (case-insensitive, ignore "I", "II" suffixes).
- **Respect Indeed's rate limits.** If the connector returns rate-limit errors, back off (wait + retry once) and report to user if persistent.
- **Don't auto-add to pipeline.** Always ask the user first (Step 6 prompt).
- **Cap token spend.** Step 4 enrichment is the biggest cost — capped at 30. If user wants more, they can re-run with refined filters.
- **Cite the source.** Every entry shown includes the Indeed `job_id` so the user can verify on indeed.com directly.

## When NOT to use this mode

- For **applying** to a known Indeed URL: use `/career-ops {url}` (auto-pipeline) directly.
- For **just one or two ad-hoc searches** (e.g., "find me Indeed jobs at Anthropic"): faster to ask Claude in chat to use the Indeed Job Search tool directly with custom parameters.
- For **non-Indeed sources** (Greenhouse, LinkedIn, university portals): use `/career-ops scan` (Playwright + WebSearch).

## Companion: linkedin-scan

LinkedIn does NOT have an official Anthropic MCP connector. For LinkedIn coverage see `/career-ops linkedin-scan` — it uses Composio's Browser Tool to navigate LinkedIn search pages (less reliable than Indeed, anti-bot risk).
