# career-ops — Session State

> Living handoff doc. Read FIRST in any new Claude Code session in this repo.

**Last updated:** 2026-05-10 (after batch apply day + UM Workday discard + state save before user /clear)

---

## TL;DR

You are running from `C:/Dev/career-ops/` — a fork of `santifer/career-ops` (`https://github.com/CodeRojoTera/career-ops`). The user (Agustín, F-1 OPT, **37 days remaining → deadline 2026-06-16**) has it fully configured + 4 custom modes added on top of upstream. Today closed strong: **4 apps submitted, 1 discarded (UM Workday — MCP friction)**. Tracker is at 36 apps with 8 Aplicado, 5 Evaluada, 21 Descartada, 2 Rechazada, 1 Respondido.

**Next session goal: massive scrape 50+ MCP-friendly opportunities + load into pipeline for daily apply runs through OPT deadline.**

---

## Today's wins (2026-05-10)

| # | Empresa | Rol | Score | Submitted via |
|---|---|---|---|---|
| 37 | Fund for the City of NY | Program Ops & Data Systems Mgr (AI for Nonprofits Sprint) | 4.6/5 | Idealist 09:35 UTC |
| 39 | Sonoran AI Consulting | AI Strategy & Research Intern | 3.0/5 | Idealist 09:38 UTC |
| 41 | Project Evident | Associate II, Data Hub | 4.0/5 | applytojob.com 09:32 UTC |
| 43 | BRITE Institute | AI Safety Research Volunteer | 3.8/5 | Idealist 09:37 UTC |

All 4 with Gmail receipt confirmation. First batch using **static Spring 2026 PDF** (per `cv.prefer_static_resume: true`) + **tailored cover letters per role**. Workflow validated end-to-end on portal types **Idealist + applytojob.com (JazzHR)**.

Discarded today:
- **#42 UM Project Coordinator (4.5/5)** — NOT a fit problem (still cap-exempt UHealth, bilingual differentiator real). Discarded due to **Chrome MCP × Workday React form friction**: 6+ failed attempts on Step 1, file_upload denied by CDP, Save and Continue button validates internally and rejects MCP-set values. Pack remains in `output/2026-05-10-um/` for future manual submit if user wants.

---

## Critical lesson learned (drives next-session priorities)

**MCP-friendly portals (auto-fill works ≥80%, file upload often works):**
- ✅ **Idealist** (Jobs / Internships / Volunteer-opportunities) — simple form, single-step
- ✅ **JazzHR / applytojob.com** — clean DOM, file upload works
- ✅ **Greenhouse** (boards.greenhouse.io, job-boards.greenhouse.io) — mostly works
- ✅ **Lever** (jobs.lever.co) — single-step usually
- ✅ **Ashby** (jobs.ashbyhq.com) — confirmed working (Mercor success April 13)
- ⚠️ **LinkedIn Easy Apply** — single-page, but anti-bot risk
- ⚠️ **Indeed Easy Apply** — single-page, similar anti-bot risk
- ⚠️ **Wellfound** (workatastartup) — usually OK

**MCP-hostile portals (avoid auto-fill, prep pack + user submits manually):**
- ❌ **Workday** (any tenant: UM `umiami.wd1`, Florida Tech `fit.wd5`, etc.) — multi-step React, file_upload CDP-denied, name field React validation rejects MCP keystrokes
- ❌ **iCIMS** — multi-step, login walls
- ❌ **SuccessFactors / PeopleFirst** (FL state) — login required + multi-step (User CAN apply manually after login as proven 2026-05-09)
- ❌ **Taleo** — legacy multi-step

**For MCP-hostile portals: skip /career-ops apply auto-fill. Generate apply pack only. User submits manually using pack §2 as guide. Faster than fighting MCP.**

---

## Tracker stats (post-2026-05-10)

```
Total: 36 apps
├── 8 Aplicado:   #26 Mercor, #27 ConnecTeam, #30 SoF Data Sci Intern, #31 SoF IT Intern,
│                 #37 FCNY, #39 Sonoran, #41 Project Evident, #43 BRITE
├── 5 Evaluada:   #18, #19, #22, #24 (4 Indeed SDR oldies),
│                 #38 Anthropic STEM Fellow (PhD gap, defer),
│                 #40 UCF AI Automation Designer (no visa sponsorship, defer)
├── 1 Respondido: #33 Jotform (Alexis Russell SDR Bootcamp — follow-up sent 2026-05-10, next check 2026-05-13)
├── 21 Descartada: 16 stale + 2 low-fit + 2 priority defers + #42 UM (MCP friction)
└── 2 Rechazada:  SoF Data Program Coordinator (Auto-DQ Mar 31), Anthropic Product Support London (May 6)
```

Pipeline: **67 entries from prior scan, 7 evaluated (4 submitted + 3 deferred/discarded). 60+ remaining un-evaluated.**

---

## Next session goal (PRIMARY)

**Massive scrape: 50+ NEW opportunities for the next 30 days of OPT push.**

Constraint: **MCP-friendly portals ONLY** (per lesson learned above). Skip Workday / iCIMS / PeopleFirst / Taleo for auto-fill — those go to manual-only path.

Categories needed:
- **Jobs (FT/PT)**: ~25-30 entries — analyst / coordinator / AI consulting / nonprofit
- **Internships**: ~10-15 entries — extends OPT runway, often visa-permissive
- **Volunteer (stackable)**: ~10-15 entries — for OPT-clock stacking strategy (per `profile.yml::cv.opt_volunteer_stacking_strategy`)

Total: 50+ entries in `data/pipeline.md` ready for evaluation + apply.

### Recommended scan strategy (next session opening prompt)

The user will paste a prompt like this — be ready:

```
Massive scrape: target 50+ entries in pipeline.md across jobs+internships+volunteer.
Constraint: MCP-friendly portals only — Idealist, JazzHR, Greenhouse, Lever, Ashby,
Wellfound, LinkedIn Easy Apply (acknowledge anti-bot risk), Indeed via connector.
SKIP all Workday tenants, iCIMS, PeopleFirst, Taleo, SuccessFactors — those don't
auto-fill and waste time.

Run in this order:
1. /career-ops indeed-scan (8-14 queries, expect 50-100 raw → 30-50 after filter)
2. /career-ops linkedin-scan (12 queries, expect anti-bot to kill some — keep what works)
3. /career-ops scan (Playwright Idealist Jobs/Internships/Volunteer + Greenhouse/Lever/Ashby boards
   already in portals.yml). SKIP the Workday-based tracked_companies (UM, FIT, Florida Tech) for
   this scan.
4. Consolidate into pipeline.md, dedupe vs applications.md, rank top 50 by composite score.
5. Show me the top 50 ranked, I pick which to /career-ops apply against (probably top 15-20
   spread over week).

Cost cap: $10-15 OK for this scope.
```

### Apply cadence after scrape

Target: **3-5 apps per day** for 7-10 days = 21-50 new applications submitted by 2026-05-20. With sustained pace, total submitted by OPT deadline = **65-100 apps**.

For each apply session:
1. Pick top 3-5 from pipeline ranked by score
2. `/career-ops apply` against each (auto-pipeline if not yet evaluated)
3. User uploads static Spring 2026 PDF + uploads tailored cover letter PDF
4. User clicks Submit
5. User says "submitted X" → tracker updates Evaluated → Aplicado

---

## Custom modes added to fork (4)

All in `modes/` (untouched by upstream auto-updater) + small SKILL.md routing additions (reverted on update — see MIGRATION-NOTES.md for cherry-pick instructions).

| Mode | Purpose | When |
|---|---|---|
| `/career-ops sync-inbox` | Read Gmail, classify recruiter emails, propose tracker status updates | Run weekly or after suspect quiet period |
| `/career-ops indeed-scan` | Discover Indeed jobs via claude.ai Indeed connector (read-only: Job Search, Job Details, Company Info, Get Resume) | Run with `scan` for full coverage |
| `/career-ops linkedin-scan` | Discover LinkedIn jobs via Composio Browser Tool — anti-bot fallback to manual | Run when wanting LinkedIn coverage |
| Step 4b in `modes/followup.md` | Auto-create Gmail drafts via MCP after followup generates text | Triggered automatically when followup runs and Gmail MCP available |

Plus: `cv.prefer_static_resume: true` config in `config/profile.yml` → all apply flows use Spring 2026 PDF instead of generated tailored CVs.

---

## Active inbound thread (acción pendiente)

**Jotform — Enterprise SDR Bootcamp** (#33 Respondido). Recruiter Alexis Russell (alexis@jotform.com) sent InMail 2026-04-30. User sent follow-up 2026-05-10 via /career-ops followup Step 4b (Gmail draft created + sent). Next check: **2026-05-13** (3 days, per "Responded → every 3 days" cadence). If no reply, send 2nd follow-up taking new angle.

---

## Open issues (non-urgent)

1. **FSU portal scan blocked**: `jobs.omni.fsu.edu` returns 404 from Playwright. Either dropear FSU del automated scan o user contacts FSU HR directly leveraging alumna status.
2. **Idealist Saved Items**: `/dashboard/saved-items` rendering via Algolia client-side, get_page_text doesn't capture. User reviews manually in browser if needed.
3. **Status normalization**: tracker mixes "Evaluated" (English) and "Evaluada" (Spanish) due to merge-tracker.mjs default. Run `node normalize-statuses.mjs` when convenient.
4. **UM Project Coordinator manual backlog**: pack ready in `output/2026-05-10-um/`, user can submit manually any day this week (~15-20 min). NOT urgent but real opportunity (4.5/5).

---

## Files / locations reference

```
C:/Dev/career-ops/
├── config/profile.yml              ← personal profile (gitignored, prefers static resume)
├── cv.md                           ← markdown CV (gitignored)
├── portals.yml                     ← scan config (gitignored)
├── data/applications.md            ← tracker (gitignored — 36 apps)
├── data/follow-ups.md              ← follow-up history (gitignored — 1 entry: Jotform)
├── data/inbox-sync-state.json      ← sync-inbox state (gitignored)
├── modes/sync-inbox.md             ← custom (tracked, survives upstream updates)
├── modes/indeed-scan.md            ← custom (tracked, survives)
├── modes/linkedin-scan.md          ← custom (tracked, survives)
├── modes/followup.md               ← upstream + Step 4b patch (tracked, REVERTS on upstream update)
├── .agents/skills/career-ops/SKILL.md  ← upstream + 4 routing additions (REVERTS on upstream update)
├── MIGRATION-NOTES.md              ← cherry-pick instructions for upstream update reverts
├── output/2026-05-10-*/            ← apply prep packs (today's batch)
└── reports/                        ← evaluation reports (auto-gen by /career-ops)
```

OneDrive paths (static resume — referenced from profile.yml):
- `C:/Users/agust/OneDrive/Documents/curriculum vitae agustin ledesma/05.spring2026/Agustin-Ledesma-Resume-Spring2026-Eng.pdf`
- `C:/Users/agust/OneDrive/Documents/curriculum vitae agustin ledesma/05.spring2026/Agustin-Ledesma-Resume-Spring2026-Esp.pdf`

---

## Cost tracking

Today's career-ops session: ~$11 (per session counter at last update). Across all scans + evals + apply prep + auto-fill: ~$3.55 in subagent token costs reported. Remaining headroom in budget for next-session massive scrape: **$10-15 budget approved**.
