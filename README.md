# AI-Career-Ops: The Multi-Agent Industrial Job Search Engine

**AI-Career-Ops** is a unified, best-of-both-worlds job application framework combining:
1. **The Multi-Agent Intelligence of `ai-job-search`**: Specialized subagents (`posting-fetcher`, `drafter`, `application-reviewer`), pre-flight security guards (`posting_guard.py`), and adversarial Factual Grounding Audits.
2. **The Industrial ATS Engine of `career-ops`**: 70+ direct ATS scrapers (Greenhouse, Lever, Workday, Ashby, etc.), Playwright HTML/CSS resume generation, interactive browser form auto-filling, Go TUI dashboard, and Next.js web application.

---

## Key Architecture & Features

```
ai-career-ops/
├── .claude/
│   ├── agents/               # posting-fetcher, application-reviewer, posting-scorer
│   └── commands/             # /apply, /gmail-sync, /interview, /outcome, /rank, /setup
├── modes/                    # Career-Ops modes: oferta.md, apply.md, pdf.md, etc.
├── providers/                # 70+ zero-token public ATS scrapers (Greenhouse, Workday, etc.)
├── tools/                    # Python toolbelt: posting_guard, verify_pdf, fetch_page, robots_check
├── config/                   # profile.yml (Candidate facts, target roles, dealbreakers)
├── cv.md                     # Master resume source of truth in plain Markdown
├── data/                     # applications.md (canonical tracker) & applications.db (SQLite cache)
├── documents/
│   ├── applications/         # Per-job archives: <company>_<role>/ (JD, tailored CV, notes)
│   └── cv/                   # Master CVs and source reference documents
├── company_research/         # Cached company research profiles
├── dashboard/                # High-speed Go Terminal UI
├── web/                      # Full Next.js / React Web Application
├── cops                      # Unified CLI launcher
├── generate-pdf.mjs          # Playwright HTML-to-PDF compiler
└── build-cv-html.mjs         # ATS HTML resume builder
```

---

## The Workflow at a Glance

1. **Discovery (`cops scan`)**:
   Searches 70+ ATS boards (Greenhouse, Ashby, Lever, Workday, etc.) with zero LLM token consumption. Results land in `data/pipeline.md`.

2. **Pre-Flight Gates**:
   Before spending AI tokens:
   - Verifies link liveness with `check-liveness.mjs`
   - Checks against your personal blacklist (`data/blacklist.md`)
   - Checks hard dealbreakers (minimum compensation floor, remote work requirements) in `config/profile.yml`
   - Sanitizes untrusted job descriptions using `tools/posting_guard.py`

3. **Multi-Agent Drafter-Reviewer Pipeline (`/apply <url>`)**:
   - **Drafter**: Evaluates fit against `cv.md` and `config/profile.yml`. Drafts role-tailored CV content and cover letter.
   - **Reviewer (`application-reviewer`)**: Fresh agent context that researches the company (cached in `company_research/<company>.json`) and conducts a strict **Factual Grounding Audit** (zero tolerance for hallucinated metrics, dates, or skills).
   - Returns structured JSON string-replacement diffs (`old_string` -> `new_string`).

4. **Playwright Resume Generation (`generate-pdf.mjs`)**:
   Converts the tailored Markdown/HTML into an ATS-formatted PDF in `output/<company>-<role>.pdf`.
   Runs `tools/verify_pdf.py` to assert text extractability and prevent page overflow.

5. **Live Form-Filling Assistant (`cops apply` / `modes/apply.md`)**:
   Opens the application in a visible Playwright browser, pre-fills application form fields based on your verified profile, highlights knock-out questions, and lets you review and click Submit.

6. **Unified Tracking & Visual Dashboards**:
   - Canonical table in `data/applications.md`
   - SQLite cache for fast querying (`data/applications.db`)
   - Interactive Go TUI: `cd dashboard && ./cops-dashboard`
   - Next.js Web App: `cd web && npm run dev`
   - Inbound email sync: `/gmail-sync` to automatically track recruiter replies, interview invites, and rejections.

---

## Quick Start Commands

```bash
# 1. Inspect Candidate Profile
cat config/profile.yml
cat cv.md

# 2. View Current Application Pipeline & Tracker Stats
node stats.mjs

# 3. Test the Security Guard on a Job Posting
python3 tools/posting_guard.py --text --json < sample_job.txt

# 4. Run the Multi-Agent Application Workflow (in Claude Code / Antigravity)
/apply https://boards.greenhouse.io/company/jobs/1234567

# 5. Launch the Web Application
cd web && npm run dev
```
