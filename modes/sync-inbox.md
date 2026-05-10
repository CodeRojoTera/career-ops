# Mode: sync-inbox -- Gmail → Tracker Status Sync

## Purpose

Read the user's Gmail (via the `claude.ai Gmail` MCP connector), find emails sent by companies in `data/applications.md`, classify each email (rejection / interview invite / response / recruiter outreach / no-op), and propose conservative status updates for the tracker. **Never apply updates without explicit user approval per row.**

This mode is the inverse of `followup`: `followup` GENERATES outbound emails; `sync-inbox` PARSES inbound emails to update tracker state.

## Inputs

- `data/applications.md` — Application tracker (source of company/role pairs to match against).
- `data/inbox-sync-state.json` — (optional, created on first use) Cache of the last sync timestamp + per-message decisions, so we don't re-process the same emails.
- Gmail MCP tools (must be enabled in this project's claude.ai connectors).
- `config/profile.yml` — Candidate's email (to filter received-from-self if any) and location (timezone for date math).

## Step 0 — Verify Gmail MCP availability

Before doing anything else, confirm Gmail tools are loaded in the current session. Look for tools matching `mcp__claude_ai_Gmail__*` (or equivalent under the user's connector naming).

If Gmail MCP is NOT available:

> "Gmail connector is not enabled in this Claude Code session. To enable it:
> 1. Visit https://claude.ai/customize/connectors
> 2. Toggle on `Gmail` (you've already completed OAuth previously)
> 3. In Claude Code, restart the session OR explicitly add the connector for this project
> Then re-run `/career-ops sync-inbox`."

Exit gracefully. Do NOT attempt to proceed.

## Step 1 — Determine sync window

Read `data/inbox-sync-state.json` if it exists. Schema:

```json
{
  "last_sync_iso": "2026-05-09T18:30:00Z",
  "decisions": {
    "<gmail_message_id>": {"decision": "rejection|interview|response|noop|skip", "applied_at": "ISO", "app_row": 28}
  }
}
```

If the file doesn't exist OR `last_sync_iso` is older than 90 days, default to scanning emails from **the last 30 days**.

If the file exists and `last_sync_iso` is recent, default to scanning emails from `last_sync_iso` minus 24h overlap (to catch any backdated arrivals).

Allow user override: if the user passed `/career-ops sync-inbox 7d` or `/career-ops sync-inbox 60d`, use that instead.

Tell the user: "Scanning Gmail from {start_date} onward..."

## Step 2 — Build the company watchlist from the tracker

Parse `data/applications.md`. Extract for every row that is NOT `Descartada`, `Rechazada`, `Oferta`:

- `app_row` (the # column)
- `company` (cleaned: drop "(Req XXXXX)" suffixes, drop trailing dashes)
- `role`
- `current_status`
- `applied_date` (Fecha column)

Filter out rows already in a terminal state. Compute the watchlist size and tell the user: "Tracking {N} active applications across {M} unique companies."

## Step 3 — Search Gmail per company

For each unique company in the watchlist, run a Gmail search using broad enough operators:

```
from:({company_domain_guess} OR {company_name}) after:{start_date}
```

Tactics for `company_domain_guess`:
- "State of Florida" → `successfactors.com OR myflorida.com OR @dms.fl.gov`
- "Florida State University" → `@fsu.edu`
- "University of Miami" → `@miami.edu OR @umiami.edu`
- Generic fallback: search for the literal company name in subject + body

If the company has a known ATS portal (career-ops `portals.yml` may help), include the ATS sender too: `noreply@greenhouse.io`, `noreply@ashbyhq.com`, `donotreply@myworkday.com`, `do-not-reply@successfactors.com`.

Use the Gmail MCP search tool. **Cap at 50 results per company** to avoid runaway scans.

**Skip already-decided messages:** if a `gmail_message_id` is already in `decisions`, skip it (don't re-classify).

## Step 4 — Classify each candidate email

For each new email retrieved, classify it into one of:

| Decision | Trigger phrases (subject OR first 200 chars of body) |
|---|---|
| **rejection** | "unfortunately", "we have decided to move forward with other candidates", "no longer being considered", "not a fit at this time", "we will not be moving forward", "automatically disqualified", "position has been filled", "decided to pursue other candidates" |
| **interview** | "interview invitation", "schedule a call", "available for a 30-minute conversation", "phone screen", "next step", "meet with our team", "calendly", "book a time" |
| **response** | "thank you for applying", "application received", "we're reviewing", "currently reviewing all applications", "we're working through applications" → **AUTO-MAP to existing status; usually no change** |
| **recruiter outreach** | InMail-style email mentioning a job from a company NOT in the tracker, offering a conversation. New opportunity. |
| **noop** | Newsletters, job alerts, marketing, unrelated comms. |
| **skip** | The email matched the company filter but content is ambiguous; let user decide manually. |

**Heuristics to reduce false positives:**

1. Verify the sender's email domain is plausible for the company (or one of the known ATS domains).
2. Only classify as `rejection` if the role title in the email matches (or strongly aligns with) a row's role. If the email doesn't reference a specific role, mark `skip`.
3. Auto-rejection emails from PeopleFirst (the State of Florida portal) often arrive within minutes of submission and explicitly say "automatically disqualified" — these are reliable.
4. LinkedIn-platform emails about applications are forwarded by LinkedIn itself (`do-not-reply@linkedin.com` or similar). Look for "Your application to {company}" in subject.

For each classified email, attempt to match it to a specific tracker row by company + role (best effort). If multiple rows match, mark `skip` and present to user.

## Step 5 — Present proposed updates

Show a dashboard, grouped by decision type, sorted by urgency (interview > rejection > recruiter outreach > skip):

```
Inbox Sync — {date} (window: {start_date} → today)

Found {N} relevant emails from {M} companies. Proposed updates:

INTERVIEW INVITES ({k}) -- act today
| App # | Company | Role | Email date | From | Subject |
| 30 | State of Florida | Data Science Internship | 2026-05-10 | recruitment@dms.fl.gov | "Phone screen invitation: Data Science Internship" |
  → Propose: Evaluada → Entrevista
  → Email link: gmail.com/...

REJECTIONS ({k})
| 32 | State of Florida - DMS Mgmt Svcs | Data Program Coordinator | 2026-03-31 | noreply@successfactors.com | "Application status update — Req 872957" |
  → Already in tracker as Rechazada (no-op, just confirming source)

RECRUITER OUTREACH ({k}) -- new opportunities
| —  | Jotform | Enterprise SDR Bootcamp | 2026-04-30 | alexis@jotform.com | "10-week SDR bootcamp" |
  → Propose: ADD new tracker row, status=Respondido, note=recruiter outreach

NO STATUS CHANGE ({k}) -- application-received confirmations
- 2026-05-09: State of Florida confirmed receipt of Data Science Internship + IT Internship submissions

SKIP / AMBIGUOUS ({k}) -- need your manual review
| App # | Company | Subject | Why skip |
| 18 | Applied ABC | "Re: Sales Development position" | Body too short, can't tell if rejection or scheduling |
```

After showing the dashboard, prompt:

> "Apply all proposed updates? [yes / no / per-row]
> 
> - `yes` → applies all interview/rejection updates + adds new outreach rows
> - `no` → exits without changes
> - `per-row` → I'll ask about each one individually"

## Step 6 — Apply updates

If user confirmed, for each applied decision:

1. **Interview invite** → update `data/applications.md`:
   - Change `Estado` column from current → `Entrevista`
   - Add note in row (or in a side `data/follow-ups.md`): "Interview invite from {sender} on {date}: {subject}"

2. **Rejection** → update `data/applications.md`:
   - Change `Estado` from current → `Rechazada`
   - If the row already says `Rechazada` (e.g., user manually marked), no-op

3. **Recruiter outreach (new opportunity)** → append a new row to `data/applications.md`:
   - `#` = next sequential number
   - `Fecha` = email date
   - `Empresa`, `Rol`, `Estado` = `Respondido`
   - Add note: "Inbound recruiter outreach from {sender}: {subject}"

4. **Skipped** → do nothing automatically; user reviews manually.

5. **Noop / response confirmation** → record decision in state file but no change to tracker.

## Step 7 — Persist state

Update `data/inbox-sync-state.json`:

```json
{
  "last_sync_iso": "<now>",
  "decisions": {
    "<msg_id_1>": {"decision": "...", "applied_at": "...", "app_row": ...},
    ...
  }
}
```

Include ALL decisions from this run (including `noop` and `skip`) so future runs don't re-process them.

## Step 8 — Summary

After applying changes, output:

```
✓ Inbox sync complete ({date})

Updates applied:
- {N} interview invites recorded
- {N} rejections confirmed
- {N} new outreach rows added

Skipped for manual review: {N}
Total emails classified this run: {N}
Next sync: just run `/career-ops sync-inbox` again — only new emails will be processed.

Action items for you:
- Reply to interview invites today (rows: ...)
- Run `/career-ops followup` to draft messages for the new outreach
```

## Rules

- **Never auto-apply without confirmation.** Even on a high-confidence rejection, show the user first.
- **Conservative default.** When in doubt, classify as `skip` and let the user decide. Better to under-apply than to mark a winning application as `Rechazada` based on a misread email.
- **Respect the state file.** Don't re-process messages already classified.
- **Cite the source.** Every proposed update must reference the email date + sender + subject so the user can verify.
- **Bilingual recipient awareness.** The user is bilingual EN/ES. Trigger phrases in both languages should be matched (e.g., Spanish: "lamentablemente", "su candidatura no ha sido seleccionada", "felicitaciones, queremos invitarlo a una entrevista").
- **Privacy.** Don't include the full email body in the dashboard output — just sender + subject + first 100 chars. The full email is one click away in Gmail if the user wants to read it.

## Trigger phrase reference (Spanish)

| Decision | Spanish phrases |
|---|---|
| rejection | "lamentablemente", "su candidatura no fue seleccionada", "hemos decidido continuar con otros candidatos", "no fue elegido para la siguiente etapa", "agradecemos su interés pero" |
| interview | "le invitamos a una entrevista", "agendar una llamada", "próximos pasos", "disponibilidad para una conversación", "queremos conocerle" |
| response | "hemos recibido su aplicación", "estamos revisando", "su candidatura está en proceso" |
