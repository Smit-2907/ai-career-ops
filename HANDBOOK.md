# AI-Career-Ops: Master Operator Handbook

> **"Apply better to fewer. Signal over volume. Evidence over keywords. A human decides. Local-first."**

Welcome to **AI-Career-Ops**, an open-source, local-first AI job search platform. This system unites:
- **The Multi-Agent Drafter-Reviewer Architecture** (from `ai-job-search`): Multi-agent separation of concerns, adversarial review, anti-hallucination audits, and prompt security guards.
- **The Industrial ATS Engine & Resume Generator** (from `career-ops`): 70+ direct ATS scrapers (Greenhouse, Lever, Ashby, Workday, etc.), zero-token scanning, Playwright HTML/CSS resume generation, live browser form filling, Go TUI, and Next.js Web UI.

---

## Table of Contents
1. [Core Principles & Privacy Architecture](#1-core-principles--privacy-architecture)
2. [Directory Anatomy](#2-directory-anatomy)
3. [Quick Setup & Configuration](#3-quick-setup--configuration)
4. [The Multi-Agent Drafter-Reviewer Workflow](#4-the-multi-agent-drafter-reviewer-workflow)
5. [Automated Job Discovery (70+ Public ATSs)](#5-automated-job-discovery-70-public-atss)
6. [Resume & PDF Compilation Engine](#6-resume--pdf-compilation-engine)
7. [Live Application Form Assistant](#7-live-application-form-assistant)
8. [Tracking, Analytics & External Sync](#8-tracking-analytics--external-sync)
9. [Command & Script Reference](#9-command--script-reference)
10. [Troubleshooting & Health Diagnostics](#10-troubleshooting--health-diagnostics)

---

## 1. Core Principles & Privacy Architecture

### The Data Contract (System vs. User Layer)
To protect your privacy and ensure safe updates, this repository strictly separates **System Files** from **User Files**:

- **System Layer (Version controlled & public-safe):**
  - Prompt modes (`modes/`), provider scrapers (`providers/`), Python tools (`tools/`), CLI scripts (`*.mjs`), dashboard (`dashboard/`), web app (`web/`), and templates (`templates/`).
- **User Layer (Strictly local & gitignored):**
  - Your personal resume (`cv.md`), profile configuration (`config/profile.yml`), personal blacklist (`data/blacklist.md`), tracker tables (`data/applications.md`), SQLite database (`data/applications.db`), application archives (`documents/applications/`), and company research caches (`company_research/`).

> [!IMPORTANT]
> The `.gitignore` file is hardened so that no personal candidate details, private email addresses, phone numbers, past employer history, generated PDF resumes, or submitted application archives can ever be committed or pushed to a remote repository.

---

## 2. Directory Anatomy

```
ai-career-ops/
├── .claude/
│   ├── agents/                   # Subagent definitions
│   │   ├── posting-fetcher.md    # Sandboxed reader + security guard
│   │   ├── posting-scorer.md     # Fit scorer against profile & dealbreakers
│   │   └── application-reviewer.md # Adversarial reviewer + Factual Grounding Audit
│   └── commands/                 # Slash commands (/apply, /gmail-sync, /interview, etc.)
├── config/
│   ├── profile.example.yml       # Template: Personal info, target roles, dealbreakers
│   └── profile.yml               # [Local] Your actual profile (gitignored)
├── cv.example.md                 # Template: Master resume structure in Markdown
├── cv.md                         # [Local] Your actual master resume (gitignored)
├── providers/                    # 70+ zero-token direct ATS scrapers
├── tools/                        # Python toolbelt
│   ├── posting_guard.py          # Sanitizes prompt injections & hidden text
│   ├── verify_pdf.py             # Asserts PDF ATS extractability & page budget
│   ├── fetch_page.py             # Safe fetcher with robots.txt check
│   └── salary_lookup.py          # Salary benchmarking utility
├── modes/                        # Career-Ops mode prompts (oferta, apply, pdf, etc.)
├── data/                         # [Local] Tracker files & derived SQLite database
│   ├── applications.md           # Canonical Markdown application tracker
│   └── applications.db           # Fast SQLite derived cache
├── documents/
│   ├── applications/             # [Local] Per-job application archives (<company>_<role>/)
│   └── cv/                       # [Local] Reference PDFs and source documents
├── company_research/             # [Local] Cached company background research
├── dashboard/                    # Interactive Go Terminal UI
├── web/                          # Next.js web application (Kanban, stats, CV studio)
├── cops                          # Unified CLI entrypoint
├── generate-pdf.mjs              # Playwright HTML-to-PDF converter
└── build-cv-html.mjs             # ATS-optimized HTML resume builder
```

---

## 3. Quick Setup & Configuration

### Step 1: Create your Master CV
Copy the template and fill in your real background:
```bash
cp cv.example.md cv.md
```
Edit `cv.md` with your truthful projects, technical skills, employment history, and education. This file is your single source of truth for facts.

### Step 2: Configure your Profile & Dealbreakers
Copy the profile template:
```bash
cp config/profile.example.yml config/profile.yml
```
Fill in:
- `candidate.full_name`, `email`, `phone`, `location`, `linkedin`, `github`
- `target_roles`: Your North Star job titles and skill domains
- `dealbreakers`: Hard rules that immediately stop wasted evaluations (e.g., minimum salary, mandatory on-site days, missing visa sponsorship)

### Step 3: Run the Health Check
Verify your local environment:
```bash
node doctor.mjs
```
This ensures Node, Python, Playwright Chromium, fonts, and configuration files are properly recognized.

---

## 4. The Multi-Agent Drafter-Reviewer Workflow

Whenever you evaluate a job posting, run:
```bash
/apply https://boards.greenhouse.io/company/jobs/1234567
```

The workflow runs through seven strict stages:

```
[Job URL / Text]
       │
       ▼
1. Pre-Flight Gates (Liveness check + Blacklist + Dealbreakers)
       │
       ▼
2. Security Gate (posting_guard.py strips prompt injections)
       │
       ▼
3. Agent 1: Drafter (Fit evaluation + tailored CV.md & Cover Letter)
       │
       ▼
4. Agent 2: Reviewer (Company research + Factual Grounding Audit)
       │
       ▼
5. Drafter Revision (Applies structured JSON diffs)
       │
       ▼
6. Playwright PDF Engine (Compiles ATS-tested PDF + verify_pdf.py)
       │
       ▼
7. Archival & Tracker Sync (data/applications.md + SQLite index)
```

### The Factual Grounding Audit
The Reviewer subagent compares every date, employer, role title, and numeric claim against your canonical `cv.md` and `config/profile.yml`. If a metric is not present in your profile, it is strictly flagged as an ungrounded claim and cannot be fabricated.

---

## 5. Automated Job Discovery (70+ Public ATSs)

`ai-career-ops` features direct scrapers for over 70 public Applicant Tracking Systems in `providers/`:
- **Supported ATSs**: Greenhouse, Ashby, Lever, Workday, BambooHR, SmartRecruiters, Personio, Recruitee, Rippling, Ultipro, PeopleSoft, RemoteOK, WeWorkRemotely, WTTJ, etc.
- **Zero-Token Architecture**: Scrapes public JSON/RSS endpoints without calling any LLM APIs, saving your tokens and budget.

Run a discovery scan:
```bash
node scan.mjs
```
Matching listings are automatically written to `data/pipeline.md` for review.

---

## 6. Resume & PDF Compilation Engine

Resumes are written in clean Markdown/HTML and compiled to PDF via Playwright Chromium:

```bash
# Compile tailored HTML to PDF:
node generate-pdf.mjs output/company-role.html output/company-role.pdf

# Verify text extractability for ATS parsers:
python3 tools/verify_pdf.py output/company-role.pdf --min-chars 100
```

### Why HTML/Playwright beats LaTeX:
- **No LaTeX dependency headaches**: No need to install multiple gigabytes of TeX Live packages (`lualatex`, `xelatex`, `fontspec`).
- **Deterministic ATS compliance**: CSS layout is pixel-perfect and the text layer is 100% extractable by parsers like Greenhouse and Workday.

---

## 7. Live Application Form Assistant

When you are ready to submit an application on an ATS website:
```bash
node apply-scope.mjs launch-options
# Or inside your AI CLI:
/career-ops apply
```
- Launches a visible Playwright browser session pointing to the application form.
- Pre-scans for **knock-out questions** (visa sponsorship, degree requirements, salary floor).
- Matches form inputs with your verified profile facts and drafts answers for each question.
- **Human-In-The-Loop**: The tool *never* clicks Submit automatically. You review all responses and submit them yourself.

---

## 8. Tracking, Analytics & External Sync

### Canonical Flat Files + Derived SQLite
- `data/applications.md` is the human-readable, git-diffable permanent record.
- `data/applications.db` is the fast derived SQLite index used by the Go TUI and Web App.
- Sync them anytime:
  ```bash
  node tracker.mjs sync
  ```

### Funnel Analytics
View your current conversion rates and response statistics:
```bash
node stats.mjs
```

### Inbound Recruiter Email Sync (`/gmail-sync`)
Scan your Gmail inbox for interview invites, coding assessments, or rejection emails:
```text
/gmail-sync
```
Proposes batch status updates for your approval before writing to `data/applications.md`.

---

## 9. Command & Script Reference

| Command / Script | Description |
| :--- | :--- |
| `node doctor.mjs` | Full environment diagnostics and health check |
| `node scan.mjs` | Runs zero-token discovery across configured ATS providers |
| `/apply <url>` | Full Multi-Agent Drafter-Reviewer application workflow |
| `node tracker.mjs sync` | Synchronizes `data/applications.md` with SQLite database |
| `node stats.mjs` | Outputs pipeline and funnel performance analytics |
| `python3 tools/posting_guard.py` | Sanitizes untrusted job descriptions |
| `python3 tools/verify_pdf.py` | Validates PDF text layer extractability |
| `cd web && npm run dev` | Launches the interactive Next.js Web Dashboard |
| `cd dashboard && ./cops-dashboard` | Launches the interactive Go Terminal UI |

---

## 10. Troubleshooting & Health Diagnostics

- **Playwright errors**: If Chromium fails to launch, run `npx playwright install chromium`.
- **Posting 403 or Login Walls**: Run `python3 tools/fetch_page.py "<URL>"` to fetch with full browser user-agent headers and check `robots.txt`.
- **Database out of sync**: If `applications.db` does not match `applications.md`, run `node tracker.mjs sync --force` to rebuild the index.
