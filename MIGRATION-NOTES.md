# Migration Notes — CodeRojoTera fork

This file documents customizations to the upstream `santifer/career-ops` repo that need attention on each upstream update.

## Custom mode: `/career-ops sync-inbox`

Added 2026-05-10 to give career-ops the ability to read Gmail (via the `claude.ai Gmail` MCP connector) and propose status updates to `data/applications.md` from inbound recruiter emails (rejections, interview invites, recruiter outreach).

**Files added (upstream's auto-updater leaves these alone):**
- `modes/sync-inbox.md` — the mode logic

**Files modified (upstream's auto-updater WILL revert these on update):**
- `.agents/skills/career-ops/SKILL.md` — 4 small additions:
  1. `argument-hint` extended with `sync-inbox`
  2. Routing table row: `| sync-inbox | sync-inbox |`
  3. Discovery menu line: `/career-ops sync-inbox → Read Gmail, classify emails...`
  4. Standalone modes list extended with `sync-inbox`

## Re-applying after an upstream update

When `update-system.mjs apply` (or the session-start update prompt) accepts a new version, the SKILL.md changes will be reverted. Re-apply with:

```bash
# 1. After update, check what was lost
git diff HEAD~1 -- .agents/skills/career-ops/SKILL.md

# 2. Re-add the 4 sync-inbox lines manually, OR cherry-pick our local commit
git log --all --oneline | grep "sync-inbox"
git cherry-pick <commit-hash-of-the-sync-inbox-routing-commit>

# 3. Re-run doctor to verify
npm run doctor

# 4. Test
claude
> /career-ops    # discovery should show sync-inbox in the menu
```

If the upstream changes also touched the routing table or argument-hint, resolve the conflict in favor of UPSTREAM's structure + add our `sync-inbox` line back in.

## Custom extension: Gmail-draft creation in `/career-ops followup`

Added 2026-05-10 to remove the copy-paste friction after `/career-ops followup` generates email drafts. The mode now has a Step 4b that uses the Gmail MCP `create_draft` tool (write tool, "Needs approval" by default) to create the drafts directly in the user's Gmail Drafts folder. The user reviews + clicks Send in Gmail (intentional safety — Gmail MCP does NOT expose Send, only create-draft).

**Files modified (upstream's auto-updater WILL revert this on update):**
- `modes/followup.md` — inserted Step 4b between current Step 4 ("Present Drafts") and Step 5 ("Record Follow-ups"). Self-contained section with conditional logic (skip silently if Gmail MCP unavailable, prompt user to opt in if available, create per-draft, do NOT mark as sent until user confirms in chat).

The Step 4b text is wrapped in a `> Custom extension added 2026-05-10 — see MIGRATION-NOTES.md` blockquote so it's visually obvious in the mode file when re-applying after an update.

**To re-apply after an upstream update of followup.md:**

```bash
# 1. Diff to see what was lost
git diff HEAD~1 -- modes/followup.md

# 2. Cherry-pick or manually re-insert the Step 4b section. Look for the
#    blockquote marker "Custom extension added 2026-05-10" — that's the
#    boundary of our addition.

# 3. Verify
npm run doctor
claude
> /career-ops followup
# Step 4b should appear in the mode file output if you ask Claude to dump
# the loaded mode, or just trust the test: a real followup invocation
# should offer to create Gmail drafts after presenting them.
```

## Custom modes: `/career-ops indeed-scan` + `/career-ops linkedin-scan`

Added 2026-05-10 to expand discovery beyond what `/career-ops scan` covers (which is heavy on cap-exempt + Greenhouse/Ashby/Lever, light on the two biggest aggregators).

### `/career-ops indeed-scan`

Uses the **`claude.ai Indeed`** MCP connector (Anthropic-managed, OAuth-completed at account level). Tools available: Job Search, Job Details, Company Information, Get Resume — all read-only.

**Files added (upstream auto-updater leaves alone):**
- `modes/indeed-scan.md` — full mode spec. 8 steps including: MCP availability check, query batch building from `portals.yml::title_filter`, dispatch to Indeed Job Search, filter pipeline (negative + citizenship + dedupe + recency), enrichment via Job Details (top 30), composite scoring, ranked presentation with `add to pipeline` / `evaluate now` / `show all` / `refine` actions.

### `/career-ops linkedin-scan`

LinkedIn has **NO official Anthropic MCP connector** as of 2026-05-10. This mode uses **Composio's BROWSER_TOOL_CREATE_TASK** (via `claude.ai Composio-JobSearch` connector) to navigate LinkedIn search pages. Anti-bot risk is real; mode handles blocks gracefully and falls back to manual workflow recommendation.

**Files added:**
- `modes/linkedin-scan.md` — full mode spec. Same 8-step pattern as indeed-scan. Differences: LinkedIn URL builder (uses `f_E`, `f_TPR`, `f_AL` query params), sequential dispatch (not parallel — anti-bot), graceful block handling (halt + suggest manual + Indeed alternative if >50% queries blocked), explicit "no auto Easy Apply" disclaimer.

### Files modified (upstream WILL revert these on update):

- `.agents/skills/career-ops/SKILL.md` — 4 edits:
  1. `argument-hint` extended with `sync-inbox | indeed-scan | linkedin-scan`
  2. Routing table extended (3 new rows)
  3. Discovery menu extended (3 new lines describing each custom mode)
  4. Standalone modes list extended (sync-inbox + indeed-scan + linkedin-scan)

### Re-applying after an upstream update

Same pattern as the prior customizations. The 3 mode files (`sync-inbox.md`, `indeed-scan.md`, `linkedin-scan.md`) survive updates. SKILL.md needs cherry-pick.

```bash
# After upstream update reverts SKILL.md, find our customization commits:
git log --all --oneline | grep -E "sync-inbox|indeed-scan|linkedin-scan|Step 4b"

# Cherry-pick or manually re-add:
#   - sync-inbox routing (commit f6602c7)
#   - followup Step 4b (commit 3cb40f7)
#   - indeed-scan + linkedin-scan routing (this commit)
```

## Static resume policy (added 2026-05-10)

User decision: **always use the iterated Spring 2026 PDF resume**, never the auto-generated tailored CVs from `/career-ops pdf` or batch scripts.

**Rationale:**
- ATS systems extract text — they don't care about visual polish.
- Recruiters DO see the visual PDF — the user's iterated version (LibreOffice-rendered) is more polished than career-ops's default Space Grotesk + DM Sans HTML template.
- Tailored CONTENT moves the needle in COVER LETTERS, not resumes. The user's resume already covers the proof points.
- Career-ops will continue to generate tailored cover letters per role (the high-leverage customization).

**Configuration (in `config/profile.yml`, gitignored personal data):**
- `cv.prefer_static_resume: true`
- `cv.static_resume_path_en: <OneDrive path to Eng PDF>`
- `cv.static_resume_path_es: <OneDrive path to Esp PDF>`
- `cv.static_resume_notes: <explanation>`

**Behavior expected from career-ops modes (read profile.yml on every invocation):**
- `apply` mode: when uploading resume, reference `static_resume_path_en` (or `_es` if form is in Spanish). Do NOT generate.
- `pdf` mode: refuse to run (or warn user) if `prefer_static_resume: true`. User must explicitly opt in if they really want a tailored PDF.
- Apply prep packs: §3 "Tailored CV emphasis notes" → still useful for the user to reference WHILE writing the cover letter, but NOT for actually generating a new PDF.

**Existing artifacts:** the 5 generated PDFs in `output/2026-05-10-*/cv-*.pdf` (and the `batch/build-tailored-cvs.mjs` script) are kept on disk for reference but should NOT be uploaded to portals. User uploads the static PDF from OneDrive instead.

This is config-only (no code change). All modes read `profile.yml::cv` section already; honoring `prefer_static_resume` is a Claude-prompt-level convention, not a hard-coded behavior.

## Long-term option

If `/career-ops sync-inbox` + followup Step 4b + `/career-ops indeed-scan` prove valuable, consider PRs to upstream `santifer/career-ops`. They are generic (MCP-connector-based, no personal data, follow career-ops's mode-spec conventions). The maintainer might accept them and the maintenance burden disappears.

`/career-ops linkedin-scan` is dicier to upstream — depends on Composio + has anti-bot risk that career-ops's brand might not want to associate with. Probably keep that one fork-only.

The static_resume policy is also worth upstreaming as an optional `cv.prefer_static_resume` flag — many career-ops users probably have polished PDFs they prefer over template-generated ones.

## Other customizations

These are personal data files (gitignored, never auto-updated, no merge concerns):

- `cv.md` — your CV
- `config/profile.yml` — your candidate profile
- `portals.yml` — your tracked companies (FSU, FL state, U Miami, Florida Tech, etc.)
- `data/applications.md` — your application tracker (migrated from old Supabase 2026-05-09)
- `modes/_profile.md` — generated by career-ops on first run from profile.yml
