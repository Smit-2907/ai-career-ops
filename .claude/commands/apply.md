# /apply - Dual-Engine Multi-Agent Job Application Workflow

You orchestrate a robust two-agent job application workflow supporting **Dual Resume Engines**:
1. **HTML / Playwright Engine (Default)**: Zero-TeX dependencies, ATS-optimized, high-speed Playwright PDF generation.
2. **LaTeX Engine (Optional / Typographer-Grade)**: Uses `moderncv` (LuaLaTeX) and `cover.cls` (XeLaTeX) for publication-quality typesetting.

The job posting is provided as `$ARGUMENTS` (either a URL or pasted text).
If `$ARGUMENTS` contains `--latex` or `config/profile.yml` specifies `resume_engine: latex`, the LaTeX engine is used; otherwise, the HTML/Playwright engine is the default.

Follow these steps **exactly in order**. Do not skip steps.

**Standing rule — write new facts back to the profile.** If the user confirms, corrects or supplies a fact not already in `cv.md` or `config/profile.yml` (a metric, project detail, tech stack, scope correction), update `cv.md` or `config/profile.yml` in the same turn. Unrecorded facts will be stripped by the Reviewer Agent's Factual Grounding Audit on future runs.

---

## Step 0: Pre-Flight Gate & Secure Ingestion

1. **Check Blacklist**: If `data/blacklist.md` exists, check the company name and URL host against it. If matched, stop immediately and ask the candidate for confirmation before proceeding.
2. **Check Dealbreakers**: Verify stated criteria (e.g., minimum compensation, mandatory 5-day on-site, visa sponsorship) against `config/profile.yml`'s dealbreakers. If broken, surface the mismatch and ask if they wish to proceed.
3. **Liveness & Ingestion (when input is a URL)**:
   - Run `node check-liveness.mjs "$ARGUMENTS"` to confirm the posting is active.
   - Dispatch `posting-fetcher` agent (`subagent_type: posting-fetcher`). It retrieves the job via portal CLI / direct fetch and runs `python3 tools/posting_guard.py` to sanitize prompt injections and hidden canary tokens.
4. **Pasted Text**: If raw text was pasted, save it to a temporary scratchpad and run `python3 tools/posting_guard.py --in <scratchpad> --text --json`.
5. Extract metadata: **company name**, **role title**, **location**, **remote status**, **salary/compensation**, and **cleaned job description**.

---

## Step 1: DRAFTER - Evaluate Fit

Read:
- `config/profile.yml`
- `cv.md` (and `cv/main_example.tex` if in LaTeX mode)

Evaluate the role across:
- **Archetype alignment**: Map against candidate target roles (e.g. AI & Backend Software Engineer, Agentic AI, FastAPI).
- **Core Skill Match**: Required vs. present in profile. Highlight any real gaps honestly (never invent missing experience).
- **Experience & Seniority Fit**: Match candidate background against stated years of experience.
- **Compensation & Logistics**: Compare with candidate expectations.
- **Fit Recommendation**: Strong Fit (score >= 4.0), Moderate Fit (3.0 - 3.9), or Skip (< 3.0).

Present the evaluation to the candidate and ask:
> "Proceed with drafting the tailored CV and cover letter for this role?"

If the user says no, stop here. If yes, proceed to Step 2.

---

## Step 2: DRAFTER - Draft Tailored Documents (Dual Engine)

1. **Target Artifact Slugs**:
   - Company slug: `<company>` (lowercase, alphanumeric + hyphens).
   - Role slug: `<role>` (lowercase, alphanumeric + hyphens).
   - Application directory: `documents/applications/<company>_<role>/`

2. **Branch by Engine**:

   ### A. HTML / Playwright Mode (Default)
   - Tailor `cv.md` experience and project bullet points for this specific JD.
   - Draft cover letter in Markdown: `documents/applications/<company>_<role>/cover_letter.md`.
   - Populate HTML template or builder JSON at `output/<company>-<role>.html`.

   ### B. LaTeX Mode (`--latex` or `resume_engine: latex`)
   - Read master references: `cv/main_example.tex` and `cover_letters/cover_example.tex`.
   - Write tailored CV: `cv/main_<company>_<role>.tex`.
   - Write tailored cover letter: `cover_letters/cover_<company>_<role>.tex`.
   - Ensure cover letter uses `cover.cls` and CV uses `moderncv` with `banking` style.
   - Keep CV strictly to 2 pages and cover letter to 1 page.

3. **Grounding Rule**: Every metric, date, company, and project must match `cv.md` or `config/profile.yml` exactly. Never fabricate.

---

## Step 3: REVIEWER - Company Research & Factual Grounding Audit

Dispatch the **`application-reviewer`** agent (`subagent_type: application-reviewer`):
Pass the drafted CV, cover letter, and job posting **inline in the prompt**.

The Reviewer will:
1. **Research the Company**: Look up company mission, recent news, engineering blog, and engineering priorities. Cache findings under `company_research/<company>.json`.
2. **Perform Factual Grounding Audit**: Compare every date, metric, company, and skill in the drafts against `cv.md` and `config/profile.yml`. Flag any ungrounded claims or hallucinations.
3. **Return Feedback in Two Parts**:
   - **Part A (Structured Edits)**: JSON array of `{ "file": "...", "old_string": "...", "new_string": "...", "reason": "..." }`.
   - **Part B (Critique)**: Narrative advice on tone, missed keywords, and company alignment.

---

## Step 4: DRAFTER - Apply Reviewer Revisions

Apply the reviewer's Part A string replacements and address any factual flags or key suggestions.

---

## Step 5: PDF Generation & Verification

### A. HTML / Playwright Engine:
1. Compile PDF:
   ```bash
   node generate-pdf.mjs "output/<company>-<role>.html" "output/<company>-<role>.pdf"
   ```
2. Verify ATS Text Extraction:
   ```bash
   python3 tools/verify_pdf.py "output/<company>-<role>.pdf" --min-chars 100
   ```

### B. LaTeX Engine:
1. Compile CV with LuaLaTeX:
   ```bash
   lualatex -interaction=nonstopmode -output-directory=cv cv/main_<company>_<role>.tex
   ```
2. Compile Cover Letter with XeLaTeX:
   ```bash
   xelatex -interaction=nonstopmode -output-directory=cover_letters cover_letters/cover_<company>_<role>.tex
   ```
3. Verify Layout and ATS Text:
   ```bash
   python3 tools/verify_layout.py cv/main_<company>_<role>.pdf --max-pages 2
   python3 tools/verify_pdf.py cv/main_<company>_<role>.pdf --min-chars 100
   ```
4. Copy compiled PDFs to `output/<company>-<role>.pdf` and `output/<company>-<role>-cover.pdf`.

---

## Step 6: Archival & Tracking

1. **Application Archive**:
   Ensure `documents/applications/<company>_<role>/` contains:
   - `job_posting.md` (exact verbatim text archived for interview prep)
   - `cover_letter.md` (or `.tex`)
   - `cv.html` (or `.tex`) and copy of compiled resume PDF
   - `outcome.md` (tracking interview rounds and notes)

2. **Register in Tracker**:
   - Append or update the application row in `data/applications.md`.
   - Sync with SQLite database:
     ```bash
     node tracker.mjs sync
     ```
   - The application is now instantly visible in the Go TUI (`dashboard/`) and Next.js Web App (`web/`)!

---

## Step 7: Live Submission Handoff (Optional)

Ask the user:
> "Your application materials are ready in `output/<company>-<role>.pdf` and archived in `documents/applications/<company>_<role>/`.
> Would you like to use the **Live Browser Assistant (`cops apply`)** to fill in the online application form now?"
