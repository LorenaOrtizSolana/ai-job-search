---
name: job-scraper
description: >
  Searches for new positions matching your profile (LinkedIn via a dedicated CLI tool, plus
  company career pages via WebSearch). Deduplicates across runs.
  Triggers on: job scrape, find jobs, search jobs, new jobs, job search, scrape jobs, /scrape
allowed-tools: Read, Write, Edit, Glob, Grep, WebFetch, WebSearch, Agent, AskUserQuestion, Bash(bun run .agents/skills/linkedin-search/cli/src/cli.ts:*)
---

# Job Scraper

---

## How It Works

This skill searches LinkedIn (via the repo's `linkedin-search` CLI tool, hitting LinkedIn's public jobs-guest API directly - no login) and company career pages (via `WebSearch`) using targeted queries based on your profile, deduplicates against previously seen jobs and the application tracker, and presents new matches with a quick fit assessment.

## Invocation

The user triggers this skill by saying things like:
- "Find new jobs"
- "Scrape for jobs"
- "Any new positions?"
- "/scrape"

Optional arguments:
- A focus area, e.g. "/scrape data science" or "/scrape geophysics" - all categories still run, but results from the matching category are weighted/prioritized when presenting
- "broad" to also include Borderline-tier locations (fully remote, outside-Germany-but-EU-hiring roles) and relax `--jobage` beyond 14 days, e.g. "/scrape broad"

---

## Execution Steps

### Step 0: Load State

1. Read `job_scraper/seen_jobs.json` (create if missing - start with `{"seen": {}}`)
2. Read `job_search_tracker.csv` to extract already-applied companies+roles
3. Read `search-queries.md` (this directory) for the search strategy

### Step 1: Search

Run **all four priority categories** from `search-queries.md` every time - there is no reduced default. If the user specified a focus area (e.g. "data science"), still run every category, but present/weight that category's results first. If the user said "broad", additionally include Borderline-tier locations and relax `--jobage`.

**LinkedIn** - use the `linkedin-search` CLI, not `WebSearch`:
```bash
bun run .agents/skills/linkedin-search/cli/src/cli.ts search -q "<keyword>" -l "<location>" --jobage 14 --format json
```
- Run each CLI keyword listed in `search-queries.md` against **each** of Frankfurt, Berlin, and Munich (Hamburg/Cologne optional) in the same pass - see the Location Filter section in `search-queries.md`. Do not wait to see if Frankfurt alone is "thin" before trying other cities.
- Keywords already have entry-level qualifiers built in (e.g. "Data Scientist Praktikum") - don't strip these back to bare role titles, that's what caused under-targeted results previously.
- `--jobage 14` matches the Date Filter - omit only if a category is thin and older postings are worth surfacing (flag date explicitly if so), or if the user said "broad."
- Keep volume reasonable (per-query, not per-page-spam) - this hits LinkedIn's public endpoint directly, so don't loop it excessively in one run. With ~4 categories x ~4 keywords x 3 locations, that's already ~48 calls - batch/parallelize where the tool allows it, and don't multiply further by adding extra locations beyond what's specified without good reason.

**Company career pages** - use `WebSearch` with the `site:` queries in `search-queries.md`.

### Step 2: Fetch & Parse

For each promising result from Step 1:
- **LinkedIn results**: use the CLI's `detail` command for the full description instead of `WebFetch`:
  ```bash
  bun run .agents/skills/linkedin-search/cli/src/cli.ts detail <id> --format plain
  ```
  The `search` step already returns title, company, location, and posting date - `detail` fills in the full description, seniority, employment type, and apply link.
- **Company career page results**: use `WebFetch` to retrieve the job posting page.
- Extract: **job title**, **company**, **location**, **posting date**, **URL**, **key requirements** (brief), **application deadline** (if listed)
- Skip if the URL/job ID or company+title combo already exists in `seen_jobs.json`
- Skip if the company+role already appears in `job_search_tracker.csv`

### Step 3: Quick Fit Assessment

For each new job, do a rapid fit check (NOT the full evaluation from `04-job-evaluation.md` - just a quick signal):

- **High match**: Role directly involves your core skills
- **Medium match**: Role is adjacent to your experience
- **Low match**: Role requires significant skills you lack

### Step 4: Deduplicate & Store

1. Add ALL fetched jobs (new and skipped) to `seen_jobs.json` with structure:
```json
{
  "seen": {
    "<url_or_company_title_key>": {
      "title": "...",
      "company": "...",
      "url": "...",
      "first_seen": "YYYY-MM-DD",
      "fit": "high/medium/low",
      "status": "new/skipped/evaluated"
    }
  }
}
```
2. Only present jobs NOT already in the seen list or tracker.

### Step 5: Present Results

Present new jobs in a table sorted by fit (high first):

```
## New Job Matches - YYYY-MM-DD

Found X new positions (Y high, Z medium, W low match).

| # | Fit | Title | Company | Location | Deadline | URL |
|---|-----|-------|---------|----------|----------|-----|
| 1 | High | ... | ... | ... | ... | [Link](...) |

### High-Match Highlights
For each high-match job, add 2-3 bullet points:
- Why it matches your profile
- Key requirements to check
- Any red flags
```

After presenting, ask:
> "Want me to evaluate any of these in detail? Just give me the number(s)."

If the user picks a number, invoke the **job-application-assistant** skill workflow (fit evaluation first, then CV + cover letter if approved).

### Step 6: Update Tracker (Optional)

If the user decides to apply to any job, add a row to `job_search_tracker.csv`.

---

## Important Rules

1. **Never fabricate job postings.** Only present jobs found via actual `linkedin-search`/`WebSearch`/`WebFetch` results.
2. **Respect deduplication.** Always check seen_jobs.json AND job_search_tracker.csv before presenting.
3. **Focus on configured geographic area.** Skip jobs that require relocation or are clearly outside commute range.
4. **Only open positions.** Skip postings with expired deadlines or those marked as closed. For company-career-page results specifically (not LinkedIn), always fetch the individual posting to confirm it's still live - these expire/get filled far more often than the LinkedIn CLI's real-time results do.
5. **Be efficient with WebFetch.** Don't fetch every company-career-page search result - use titles and snippets to pre-filter before fetching. This doesn't apply to the LinkedIn CLI's `detail` command, which is cheap and reliable.
6. **Keep LinkedIn CLI volume reasonable.** It hits LinkedIn's public jobs-guest endpoint directly - per the tool's own notice, this is personal-use only and against LinkedIn's ToS if done at bulk/commercial volume. Running the handful of keyword searches in `search-queries.md` per `/scrape` invocation is fine; don't loop it into dozens of calls in one run.
7. **Parallel searches.** Use the Agent tool or parallel `WebSearch` calls to speed up the company-career-page search phase. The LinkedIn CLI itself already retries rate limits internally - run its searches sequentially rather than in a large parallel burst.
