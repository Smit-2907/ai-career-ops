# /apply - Multi-Agent Job Application Workflow (Hybrid Engine)

You orchestrate a robust two-agent job application workflow combining the **Multi-Agent Drafter-Reviewer loop** with the **Playwright ATS Resume Engine & Live Application Assistant**.

The job posting is provided as `$ARGUMENTS` (either a URL or pasted text).

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
- `cv.md`

Evaluate the role across:
- **Archetype alignment**: Map against candidate target roles (e.g. AI & Backend Software Engineer, Agentic AI, FastAPI).
- **Core Skill Match**: Required vs. present in `cv.md`. Highlight any real gaps honestly (never invent missing experience).
- **Experience & Seniority Fit**: Match candidate background against stated years of experience.
- **Compensation & Logistics**: Compare with candidate expectations.
- **Fit Recommendation**: Strong Fit (score >= 4.0), Moderate Fit (3.0 - 3.9), or Skip (< 3.0).

Present the evaluation to the candidate and ask:
> "Proceed with drafting the tailored CV and cover letter for this role?"

If the user says no, stop here. If yes, proceed to Step 2.

---

## Step 2: DRAFTER - Draft Tailored CV + Cover Letter

1. **Target Artifact Paths**:
   - Company slug: `<company>` (lowercase, alphanumeric + hyphens).
   - Role slug: `<role>` (lowercase, alphanumeric + hyphens).
   - Application directory: `documents/applications/<company>_<role>/`
   - Tailored CV HTML/JSON: `output/<company>-<role>.html` (or `documents/applications/<company>_<role>/cv.html`)
   - Cover letter Markdown: `documents/applications/<company>_<role>/cover_letter.md`

2. **Tailor CV**:
   - Reorder experience and project bullet points so the strongest evidence matching the JD appears first.
   - Highlight matching technologies (e.g., Python, FastAPI, BERT, Pydantic, PostgreSQL, Agentic Workflows).
   - Inject JD keywords naturally where truthful.
   - **Grounding Rule**: Every metric, date, company, and project must match `cv.md` or `config/profile.yml` exactly. Never fabricate.

3. **Draft Cover Letter**:
   - Address the specific hiring team and role.
   - Explain why the candidate's verified projects and technical experience directly solve the employer's needs.
   - Keep to ~1 page, confident, clear, and professional.

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

## Step 5: PDF Generation & ATS Extraction Verification

1. **Build PDF with Playwright Engine**:
   Run:
   ```bash
   node generate-pdf.mjs "output/<company>-<role>.html" "output/<company>-<role>.pdf"
   ```
   (Or run the template builder `node build-cv-html.mjs` followed by `generate-pdf.mjs`).

2. **Verify ATS Extraction & Page Count**:
   Run:
   ```bash
   python3 tools/verify_pdf.py "output/<company>-<role>.pdf" --min-chars 100
   ```
   This verifies that the PDF text layer is fully extractable by modern ATS systems and that pages do not overflow.

---

## Step 6: Archival & Tracking

1. **Application Archive**:
   Ensure `documents/applications/<company>_<role>/` contains:
   - `job_posting.md` (exact verbatim text archived for interview prep)
   - `cover_letter.md`
   - `cv.html` and copy of `output/<company>-<role>.pdf`
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
> Would you like to use the **Live Browser Assistant (`/cops apply`)** to fill in the online application form now?"

If the user wants live form-filling, trigger `modes/apply.md` to launch Playwright and guide form completion.
