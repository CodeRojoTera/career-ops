# Mode: followup -- Follow-up Cadence Tracker

## Purpose

Track follow-up cadence for active applications. Flag overdue follow-ups, extract contacts from notes, and generate tailored follow-up email/LinkedIn drafts using report context.

## Inputs

- `data/applications.md` — Application tracker
- `data/follow-ups.md` — Follow-up history (created on first use)
- `reports/` — Evaluation reports (for context in drafts)
- `config/profile.yml` — User profile (name, identity)
- `cv.md` — CV for proof points in drafts

## Step 1 — Run Cadence Script

Execute:

```bash
node followup-cadence.mjs
```

Parse the JSON output. It contains:

| Key | Contents |
|-----|----------|
| `metadata` | Analysis date, total tracked, actionable count, overdue/urgent/cold/waiting counts |
| `entries` | Per-application: company, role, status, days since application, follow-up count, urgency, next follow-up date, extracted contacts, report path |
| `cadenceConfig` | Cadence rules (applied: 7 days, responded: 3 days, interview: 1 day) |

If no actionable entries, tell the user:
> "No active applications to follow up on. Apply to some roles first with `/career-ops` and come back when they're aging."

## Step 2 — Display Dashboard

Show a cadence dashboard sorted by urgency (urgent > overdue > waiting > cold):

```
Follow-up Cadence Dashboard — {date}
{N} applications tracked, {N} actionable

| # | Company | Role | Status | Days | Follow-ups | Next | Urgency | Contact |
```

Use visual indicators:
- **URGENT** — respond within 24 hours (company replied)
- **OVERDUE** — follow-up is past due
- **waiting (X days)** — on track, follow-up scheduled
- **COLD** — 2+ follow-ups sent, suggest closing

## Step 3 — Generate Follow-up Drafts

For each **overdue** or **urgent** entry only:

1. Read the linked report (`reportPath` from JSON) for company context
2. Read `cv.md` for proof points
3. Read `config/profile.yml` for candidate name and identity

### Email Follow-up Framework (first follow-up, followupCount == 0)

Generate a 3-4 sentence email:

1. **Sentence 1:** Reference the specific role + when you applied. Be specific — mention the company name and role title.
2. **Sentence 2:** One concrete value-add from the report's Block B match or a proof point from cv.md. Quantify if possible.
3. **Sentence 3:** Soft ask + availability. Offer a specific time window ("this week" or "next Tuesday").
4. **Sentence 4 (optional):** Brief mention of a relevant recent project or achievement.

**Rules:**
- Professional but warm, NOT desperate
- **NEVER** use "just checking in", "just following up", "touching base", or "circling back"
- Lead with value, not with the ask
- Reference something specific to THAT company (from report Block A)
- Keep under 150 words
- Include a subject line
- Use the candidate's name from `config/profile.yml`

**Example tone:**
> Subject: Re: Senior PHP/Laravel Developer — IxDF
>
> Hi [contact name or "team"],
>
> I submitted my application for the Senior PHP/Laravel Developer role on April 7th. I wanted to share that my production Laravel app (Barbeiro.app — 120 models, 315 API endpoints, full test suite) closely mirrors the TDD-driven culture described in the posting.
>
> I'd love to discuss how my 15 years of PHP experience and hands-on AI tooling workflow could contribute to IxDF's platform. Would any time this week work for a brief conversation?
>
> Best,
> [Name]

### LinkedIn Follow-up (if no email contact found)

Reuse the contacto framework: 3 sentences, 300 character max.
- Hook specific to company → proof point → soft ask
- Suggest the user run `/career-ops contacto {company}` to find the right person first

### Second Follow-up (followupCount == 1)

Shorter than first (2-3 sentences). Take a **new angle**:
- Share a relevant insight, article, or project update
- Don't repeat the first follow-up's content
- Still reference the role specifically

### Cold Application (followupCount >= 2)

Do NOT generate another follow-up. Instead suggest:
> "This application has had {N} follow-ups with no response. Consider:
> - Updating status to `Discarded` if the role seems filled
> - Trying a different contact via `/career-ops contacto`
> - Keeping in `Applied` status but deprioritizing"

## Step 4 — Present Drafts

For each draft, show:

```
## Follow-up: {Company} — {Role} (#{num})

**To:** {email or "No contact found — run `/career-ops contacto` first"}
**Subject:** {subject line}
**Days since application:** {N}
**Follow-ups sent:** {N}
**Channel:** Email / LinkedIn

{draft text}
```

### Step 4b — (Optional) Create drafts directly in Gmail via MCP

> **Custom extension added 2026-05-10 — see `MIGRATION-NOTES.md` in repo root.**
> Survives only inside this fork; upstream auto-updater reverts this section.

After presenting all drafts in Step 4, check if the **Gmail MCP connector** is available in the current Claude Code session. Look for tools matching `mcp__claude_ai_Gmail__*` (specifically the create-draft tool, often `GMAIL_CREATE_DRAFT` or similar — the exact name depends on the user's connector configuration).

**If Gmail MCP is NOT available:** skip this step silently and continue to Step 5 (the user will manually copy-paste from the dashboard into Gmail).

**If Gmail MCP IS available:** prompt the user:

> "I can create these {N} drafts directly in your Gmail Drafts folder so you only need to review and click Send. Want me to?
>
> - `yes` — create all {N} drafts
> - `no` — skip, I'll copy-paste manually
> - `select` — let me pick which ones (only useful when {N} >= 3)"

If the user picks `yes` or `select`, for each chosen email-channel draft (skip LinkedIn-channel drafts — those need to be sent manually via LinkedIn UI):

1. Call the Gmail MCP create-draft tool with:
   - `to`: contact email from the draft (if "No contact found", skip this draft and tell the user to run `/career-ops contacto` first)
   - `subject`: subject line from the draft
   - `body`: draft text (preserve newlines)
   - `cc`: only if the user explicitly mentioned a cc in this session

2. Confirm each draft creation in the chat:
   ```
   ✓ Draft created in Gmail
     To: {to}
     Subject: {subject}
     {gmail-draft-link if the MCP returned one}
   ```

3. After all drafts created, summarize:
   ```
   Created {N} drafts in your Gmail. Open https://mail.google.com/mail/u/0/#drafts to review and send.

   Important: I did NOT send any of these — only created drafts. You must
   click Send in Gmail to actually deliver them. This is intentional safety
   (email is irreversible past the 30-second Undo Send window).
   ```

4. Do **NOT** mark these as "sent" in Step 5 yet. Wait for the user to come back and confirm in chat: "I sent the Jotform one" → then Step 5 records it. Drafts created != follow-up sent.

**Edge cases:**

- **No contact email** for a draft (only LinkedIn): skip Gmail draft creation for that one, tell the user "{Company}: no email contact found — send via LinkedIn UI as drafted above, OR run `/career-ops contacto {company}` to find an email first."
- **Multiple recipients suggested**: pick the primary one only. Don't fan-out to a list.
- **Send tool NOT exposed by Gmail MCP**: this is by design (Anthropic's safety model only exposes create-draft, not send). Do not attempt to find a workaround. The "review + send" handoff to the user is the intended UX.
- **Permission error or rate limit**: report the failure clearly, ask the user to manually create the missing drafts in Gmail.

## Step 5 — Record Follow-ups

After the user reviews and says they've sent a follow-up, record it:

1. If `data/follow-ups.md` doesn't exist, create it:
   ```markdown
   # Follow-up History

   | # | App# | Date | Company | Role | Channel | Contact | Notes |
   |---|------|------|---------|------|---------|---------|-------|
   ```

2. Append a row with:
   - `#` = next sequential number in the follow-ups table
   - `App#` = application number from tracker
   - `Date` = today's date
   - `Company` = company name
   - `Role` = role title
   - `Channel` = Email / LinkedIn / Other
   - `Contact` = who it was sent to
   - `Notes` = brief note (e.g., "First follow-up, referenced Barbeiro.app")

3. Optionally update the Notes column in `data/applications.md` with "Follow-up {N} sent {YYYY-MM-DD}"

**IMPORTANT:** Only record follow-ups the user confirms they actually sent. Never record a draft as sent.

## Step 6 — Summary

After showing all drafts, summarize:

> **Follow-up Dashboard** ({date})
> - {N} applications being tracked
> - {N} overdue — drafts generated above
> - {N} urgent — respond today
> - {N} waiting — next follow-up dates shown
> - {N} cold — consider closing
>
> Review the drafts above and tell me which ones you've sent so I can record them.

## Cadence Rules Reference

| Status | First follow-up | Subsequent | Max attempts |
|--------|----------------|------------|-------------|
| Applied | 7 days after application | Every 7 days | 2 (then mark cold) |
| Responded | 1 day (urgent reply) | Every 3 days | No limit |
| Interview | 1 day after (thank-you) | Every 3 days | No limit |

These defaults can be overridden via `node followup-cadence.mjs --applied-days N`.
