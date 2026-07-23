# Search Queries for Job Scraper

## Search Sites

Primary (Germany-focused, no Danish-specific portals - candidate is based in Frankfurt am Main, Germany):
- **linkedin.com/jobs** - via the repo's dedicated `linkedin-search` CLI tool (`.agents/skills/linkedin-search/`), which hits LinkedIn's public jobs-guest API directly (no login, real current listings, recency-filterable). Do **not** use `WebSearch` with `site:linkedin.com/jobs` - it mostly surfaces generic aggregator/category pages and stale cached postings instead of individual listings.
- Google site-searches (via `WebSearch`) against known target companies' career pages - still the right tool for this, since those pages aren't on LinkedIn

Secondary:
- **xing.com** - popular in the DACH region, can be added later if useful
- **stepstone.de**, **indeed.de** - broader German-market job boards, can be added later if useful

## Query Categories

Queries are grouped by priority. Each entry is a `linkedin-search` CLI keyword (`-q`) to run against the locations in **Location Filter** below. Use `--jobage 14` to match the Date Filter. Company-career-page searches remain `WebSearch` queries.

**All four categories below run every time by default** - there is no "top 3 only" restriction. The old default of skipping Priority 4 was a mistake: Priority 4 carries the entry-level/working-student-qualified keywords, which are exactly the terms most likely to surface genuine Praktikum/Werkstudent postings rather than senior/full-time roles that then need filtering out downstream. If the user specifies a focus area (e.g. "/scrape cryptography"), still run all four, but weight/prioritize results from the matching category first when presenting.

**Keywords are written with entry-level qualifiers built in**, not as bare role titles - a search for just "Data Scientist" returns mostly senior/full-time roles that waste a fetch-and-discard cycle. Searching for "Data Scientist Praktikum" or "Junior Data Scientist" directly targets the segment this candidate is actually eligible for.

### Priority 1: Data Scientist / ML Engineer

These match the strongest and most desired career direction.

CLI keywords (`-q`):
```
Data Scientist Praktikum
Junior Data Scientist
Machine Learning Engineer Praktikum
Werkstudent Machine Learning
```

### Priority 2: Data Engineer

These match direct experience from the Deutsche Bundesbank/AnaCredit internship.

CLI keywords (`-q`):
```
Data Engineer Praktikum
Werkstudent Data Engineer
Junior Data Engineer
Data Quality Praktikum
```

### Priority 3: Cryptography / Security-focused Data roles

Adjacent roles building on the lattice-based cryptography research background.

CLI keywords (`-q`):
```
Praktikum cryptography
Werkstudent security engineer
Junior applied cryptography
```

### Priority 4: Broader Technical / Consulting

Wider net for general technical or entry-level roles suited to a student finishing a degree. No longer conditional - runs every time alongside 1-3.

CLI keywords (`-q`):
```
Werkstudent data analyst Python
working student data science
Praktikum data analytics
graduate program data
```

### Company career pages (WebSearch, not the CLI)

For target companies without reliable LinkedIn presence, or to catch postings LinkedIn doesn't carry, still use `WebSearch` with `site:<company-careers-domain>` queries, e.g. `site:careers.deutsche-boerse.com Praktikum data`. Verify status by fetching the specific posting before presenting - see Important Rules in `SKILL.md` about expired/filled postings.

## Target Sectors & Companies

- **Banking / Financial services:** central banks (e.g. Deutsche Bundesbank, ECB), commercial banks (Deutsche Bank, Commerzbank, DZ Bank), fintech companies
- **Tech:** companies hiring Data Scientists/ML Engineers broadly (open to a wide range of tech employers)
- **Security / Cryptography-focused firms:** security consultancies, research labs, and companies with a cryptography/security angle

## Location Filter

- **Ideal:** Frankfurt am Main and surrounding area
- **Acceptable:** Other major German cities (Berlin, Munich, Hamburg, Cologne) if remote/hybrid flexibility is offered
- **Borderline:** Fully remote roles based outside Germany but hiring EU-wide
- **Too far:** Roles requiring relocation outside Germany/EU with no remote option

**Search all of Ideal + Acceptable in the same pass, every run** - do not wait to see if Frankfurt alone returns "few enough" results before trying other cities; that threshold is subjective and was likely causing under-search. Concretely, run each CLI keyword against `Frankfurt, Hesse, Germany`, `Berlin, Germany`, and `Munich, Bavaria, Germany` at minimum (Hamburg/Cologne optional if time allows). Tag results from non-Frankfurt locations clearly when presenting so the user can weigh the commute/relocation trade-off themselves, but don't withhold them from the search pass itself.

## Date Filter

Only include jobs posted within the last 14 days, or with an application deadline that has not yet passed. If a posting date cannot be determined, include it but flag as "date unknown".

## Adapting Queries

If the user specifies a focus area, select queries from the matching category and also generate 2-3 custom queries for that focus. For example:
- "/scrape cryptography" -> Priority 3 queries + custom focus-specific queries
