# Mode: linkedin-scan -- LinkedIn Job Discovery via Composio Browser Tool

## Purpose

Discover LinkedIn job postings matching the candidate's archetype. **LinkedIn has NO official Anthropic MCP connector** — this mode uses **Composio's BROWSER_TOOL_CREATE_TASK** to navigate LinkedIn search pages with browser automation.

**Important caveats:**
- LinkedIn aggressively detects browser automation. Composio Browser Tool uses some stealth but may fail or get throttled.
- LinkedIn Easy Apply automation is NOT in scope (would require login + cookie management + ToS-questionable bypasses).
- This mode is best for **discovery** only. For each found URL, run `/career-ops {url}` (auto-pipeline) to evaluate, then **manually** apply on linkedin.com.

If LinkedIn scraping fails repeatedly: fall back to the **manual workflow**:
1. User browses linkedin.com/jobs in their browser with their normal session.
2. User pastes interesting job URLs to `/career-ops {url}` for auto-pipeline.
3. User clicks Easy Apply manually on LinkedIn.
4. `/career-ops sync-inbox` catches inbound LinkedIn-forwarded emails (recruiter responses) and proposes tracker updates.

## Inputs

- `portals.yml` — `title_filter.positive` and `negative` lists (drives query construction).
- `data/applications.md` — dedupe target.
- `config/profile.yml` — candidate location/preferences.
- **Composio MCP tools** — must be enabled (the user's `claude.ai Composio-JobSearch` connector). Look for tools matching `mcp__claude_ai_Composio-JobSearch__COMPOSIO_*`. The mode uses `COMPOSIO_MULTI_EXECUTE_TOOL` to invoke `BROWSER_TOOL_CREATE_TASK` (a Composio toolkit).

## Step 0 — Verify Composio MCP availability

Confirm Composio tools are loaded. Look for `mcp__claude_ai_Composio-JobSearch__COMPOSIO_*` (the user's connector exposes COMPOSIO_GET_TOOL_SCHEMAS, COMPOSIO_SEARCH_TOOLS, COMPOSIO_MULTI_EXECUTE_TOOL, COMPOSIO_REMOTE_BASH_TOOL, COMPOSIO_REMOTE_WORKBENCH).

If NOT available:

> "Composio-JobSearch connector is not enabled for this Claude Code session. Enable it via https://claude.ai/customize/connectors then restart this session. Alternatively, if you don't want to set up Composio, use the manual workflow: browse linkedin.com/jobs, paste interesting URLs to `/career-ops {url}`."

Exit gracefully.

## Step 1 — Build LinkedIn search URLs from `portals.yml`

LinkedIn Jobs search URL pattern:

```
https://www.linkedin.com/jobs/search/?keywords={keywords}&location={location}&f_E={experience}&f_TPR={posted}&f_AL=true
```

Parameters:
- `f_E=2`: Entry level (other values: 1=internship, 3=associate, 4=mid-senior, 5=director, 6=executive)
- `f_TPR=r604800`: Past week (values: r86400=24h, r604800=7d, r2592000=30d)
- `f_AL=true`: Easy Apply only (drop if you also want external apply)
- `f_WT=2`: Remote (1=onsite, 3=hybrid)

Build 6-8 URLs from grouped keywords (same groupings as `indeed-scan.md`):

| Group | URL pattern (location=United States, f_E=2, f_TPR=r604800) |
|---|---|
| Analyst entry | `?keywords=business%20analyst%20OR%20operations%20analyst%20OR%20data%20analyst%20OR%20innovation%20analyst&...` |
| AI/Tech entry | `?keywords=AI%20consultant%20OR%20AI%20strategy%20OR%20junior%20LLM%20engineer%20OR%20AI%20fellow&...` |
| Coordinator entry | `?keywords=program%20coordinator%20OR%20project%20coordinator%20OR%20research%20coordinator&...` |
| Sales entry | `?keywords=BDR%20OR%20SDR%20OR%20business%20development%20representative&...` |
| Bilingual | `?keywords=bilingual%20spanish%20OR%20latin%20america&...` |
| Fellowship | `?keywords=fellowship%20OR%20innovation%20fellow%20OR%20AI%20for%20good&...` |

For each group, build BOTH:
- `f_E=2` (entry level) URL
- `f_E=1` (internship) URL  

= up to 12 search URLs total.

## Step 2 — Dispatch Composio Browser Tool batch

For each URL, invoke `COMPOSIO_MULTI_EXECUTE_TOOL` with toolkit `BROWSER_TOOL_CREATE_TASK` and prompt:

```text
Navigate to {url} and extract the first 25 job listings visible on the LinkedIn Jobs search results page.

For each listing extract:
- title (job title)
- company (company name)
- location (city/state, "Remote", or "Hybrid - {city}")
- posted_date ("3 days ago", "1 week ago", or absolute)
- easy_apply (boolean — true if "Easy Apply" badge visible)
- promoted (boolean — true if "Promoted" badge visible)
- posting_url (full LinkedIn URL with /jobs/view/{id}/)
- short_description (first 150 chars of the snippet shown in the card)

Return as JSON: {"listings": [...], "search_url": "{url}"}

Do not click into individual listings. Just scrape the search results page.
If LinkedIn shows a sign-in wall or anti-bot challenge, return {"error": "blocked", "url": "{url}"} and stop.
```

**Run sequentially** (not parallel) to reduce LinkedIn anti-bot triggering. ~30-60 seconds per query, ~6-12 minutes total.

## Step 3 — Handle anti-bot blocks gracefully

If a query returns `{"error": "blocked", ...}`, **do NOT retry immediately**. LinkedIn anti-bot ratchets up after repeated attempts. Instead:

1. Report to user: "LinkedIn blocked the {group} query. Skipping. {N} other queries succeeded."
2. Continue with remaining queries.
3. After all queries, summarize what was blocked + suggest manual alternatives.

If MORE than half the queries got blocked, halt the scan and tell the user:

> "LinkedIn is actively rate-limiting / blocking this scan ({N}/{M} queries blocked). Recommendation:
> 1. Try again in 24 hours (anti-bot decays).
> 2. Or switch to manual flow: browse linkedin.com/jobs in your browser, paste interesting URLs to `/career-ops {url}`.
> 3. Or use the Indeed connector via `/career-ops indeed-scan` instead — Indeed's Anthropic-managed connector is faster and doesn't get blocked."

## Step 4 — Apply filter pipeline

Same as `indeed-scan.md::Step 3`:
1. Negative filter (drop senior/staff/etc).
2. Citizenship filter (drop "US citizens only").
3. Dedupe vs `data/applications.md`.
4. Drop `promoted=true` results (low signal — paid placement, often not actually open).
5. Drop posted >14 days ago.

## Step 5 — Rank by composite score

Same weighting as `indeed-scan.md::Step 5`. Plus a bonus signal:

- `+0.05` if `easy_apply=true` (lower friction to apply)

## Step 6 — Present ranked results

Same UX as `indeed-scan.md::Step 6`:

```
LinkedIn scan — {date} (window: posted last 7 days)

{N raw} → {N after filters} → showing top 20

| # | Score | Company | Role | Loc | Easy Apply | Indicators |
| 1 | 0.82  | Anthropic | Recruiting Coord (Contract) | London | ✓ | bilingual, contract |
...

(plus N hidden 0.4-0.6 stretch entries: type `show all`)
{N blocked queries — try again later or refine}
```

After table:

> "What now?
> - `add top N to pipeline`
> - `evaluate #1, #3` (runs `/career-ops {url}` per entry — note: WebFetch may also hit anti-bot; if so, fall back to Composio BROWSER_TOOL on the specific URL)
> - `apply manually` (career-ops can't auto-apply LinkedIn — opens the URL for you to click Easy Apply manually + generates suggested form responses you copy-paste)
> - `done`"

## Step 7 — Persist scan history (optional)

Same as `indeed-scan.md::Step 7`, with `scan_method=composio_browser_linkedin`.

## Rules

- **Don't fight LinkedIn anti-bot.** If blocked, suggest manual workflow rather than escalating to undetected-chromedriver / runAiBot-style tactics (those risk LinkedIn account ban).
- **Don't claim Easy Apply automation.** Career-ops cannot auto-apply on LinkedIn. Period. Surface URL + draft responses, user clicks Easy Apply themselves.
- **Privacy.** Composio Browser Tool runs in a Composio-managed sandbox, not the user's local Chrome. Don't try to share user's LinkedIn cookies (they're in user's local browser, not accessible to Composio).
- **Conservative spending.** Each Composio Browser task = ~$0.05-0.20. 12 queries = $1-3 max per scan. Budget OK.

## When NOT to use this mode

- For **Indeed**: use `/career-ops indeed-scan` (better, official connector).
- For **non-LinkedIn manual paste**: just `/career-ops {url}` directly.
- If LinkedIn is consistently blocking: switch to Indeed coverage or just manual browsing of linkedin.com/jobs in user's normal browser session.

## Companion modes

- `/career-ops indeed-scan` — Indeed equivalent (more reliable).
- `/career-ops scan` — multi-portal Playwright + WebSearch (covers FSU/UM/PeopleFirst/Idealist/etc.).
- `/career-ops sync-inbox` — catches inbound LinkedIn-forwarded recruiter emails via Gmail MCP.
