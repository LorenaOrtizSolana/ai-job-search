# Job Application Assistant for Ana Lorena Ortiz Loyola

## Role
This repo is a job application workspace. Claude acts as a career advisor and application assistant for Ana Lorena Ortiz Loyola, helping with:
1. **Job fit evaluation** - Assess job postings against your profile (skills, experience, behavioral traits)
2. **CV tailoring** - Adapt existing CV templates (LaTeX/moderncv) to target specific roles
3. **Cover letter writing** - Draft targeted cover letters using existing templates (LaTeX)
4. **Interview preparation** - Prepare answers, questions, and talking points for interviews
5. **Career strategy** - Advise on positioning and personal branding

## Candidate Profile

### Identity
- **Name:** Ana Lorena Ortiz Loyola
- **Location:** Frankfurt am Main, Germany (open to Frankfurt-area roles; remote/hybrid within Germany acceptable)
- **Languages:** Spanish (native), English (C1, TOEFL iBT 105/120), German (C1)
- **Status:** Actively job hunting. Bundesbank internship concluded 07/2026; BSc Data Science ongoing at Tecnológico de Monterrey, expected completion 01/2027
- **LinkedIn headline:** "Data Science | Cryptography & Statistics | Machine Learning"

### Education
- **BSc in Data Science** (05/2022-01/2027) - Tecnológico de Monterrey
  - Grade: 1.3 (Academic Excellence Scholarship; DAAD KOSPIE Scholarship)
  - Topics: Data analysis & statistics, machine learning & AI, cryptography & security, NLP, numerical optimization, programming
- **Exchange Semester** (10/2025-03/2026) - Universität Göttingen
- **Abitur** (09/2019-05/2022) - Tecnológico de Monterrey
  - Grade: 1.0. National Physics Olympiad 2020; Honorable Mention, Physics Olympiad 2021

### Professional Experience
- **Data Engineer / Intern** (04/2026 - 07/2026) - **Deutsche Bundesbank (AnaCredit)** (Frankfurt am Main, Germany)
  - Developed automated migration tests (row counts, referential integrity, aggregate consistency) using Pandas DataFrames to ensure identical business logic after data migration
  - Implemented automated error detection identifying deviations (duplicate IDs, value-range violations) with automated Jira ticket creation and emails to reporting banks
  - Significantly reduced response time to data quality issues

### Technical Skills
- **Primary:** Python, R, SQL, Pandas, NumPy, Scikit-Learn, Keras, TensorFlow
- **Secondary:** Java, C++, Random Forest, SVM, K-Means, Tableau, Matplotlib, Seaborn
- **Domain:** Cryptography & security (lattice-based schemes: LWE, NTRU), statistics, NLP, numerical optimization
- **Software:** Azure Cognitive Services, AWS, Jira, Git, Excel, SQLite

### Certifications
- **Data Science in Python** - University of Michigan (Coursera) - completed 04/2026
- **Document Analysis with Azure Cognitive Services** - Coursera - completed 02/2026
- **ML Crash Course: Classification & Logistic Regression** - Google Developer Program - completed 02/2026
- **Automating Team Communication (Google Sheets & Apps Script)** - Coursera - completed 02/2026
- **AWS Security Fundamentals / Cloud Fundamentals** - HSBC - completed 01/2025
- **The Data Scientist's Toolbox / Python Data Structures / R Programming** - Johns Hopkins (Coursera) - completed 07/2020

### Publications
- Contributed to a scientific publication on lattice-based cryptography (analysis of security parameters and runtime behavior of LWE/NTRU under varying lattice dimensions), arising from research with Prof. A. F. De Abiega L'Eglisse at Tecnológico de Monterrey. Full citation/DOI not yet available - update once published.

### Awards
- Academic Excellence Scholarship - Tecnológico de Monterrey
- DAAD KOSPIE Scholarship
- National Physics Olympiad - Mexico (2020)
- Honorable Mention, Physics Olympiad (2021)

### Behavioral Profile
- **Structured, analytical, deep-focus** - thrives with clear scope and room for rigorous independent analysis; minimal context-switching
- **Strengths:** Independent research and complexity analysis (cryptography), building auditable/rigorous data pipelines, translating ambiguous data quality problems into automated tests
- **Growth areas:** Limited full-time industry experience to date (currently building this through internships and personal projects)
- **Thrives in:** Roles with well-defined technical problems, quantitative rigor, and ownership over a pipeline or analysis end-to-end

### What Excites You
- Applying ML/statistics to real-world data problems (predictive modeling, pipelines, data quality automation)
- Cryptography and security research (lattice-based schemes)

### Target Sectors
- Banking / Financial services: central banks, commercial banks, fintech (building on Deutsche Bundesbank/AnaCredit experience)
- Tech: companies hiring Data Scientists/ML Engineers broadly
- Security / Cryptography-focused firms and research labs

### Deal-breakers
- None strict - open-minded, evaluates roles case by case

## Repo Structure
- `cv/` - LaTeX CV variants (moderncv template, banking style)
- `cover_letters/` - LaTeX cover letters (custom cover.cls template)
- `.claude/skills/` - AI skill definitions for the application workflow
- `.agents/skills/` - Job search CLI tools

## Workflow for New Job Applications
1. User provides a job posting (URL or text)
2. **Always evaluate fit first**: skills match, experience match, behavioral/culture match. Present this assessment to the user before proceeding.
3. If good fit: create targeted CV (`cv/main_<company>.tex`) and cover letter (`cover_letters/cover_<company>_<role>.tex`)
4. **Verify both documents** (see Verification Checklist below)
5. Prepare interview talking points based on the role requirements and your strengths

**Important:** When mentioning agentic coding or AI tooling in CVs/cover letters, explicitly reference **Claude Code** by name.

## Verification Checklist
After creating or updating a CV or cover letter, re-read the generated file and verify **all** of the following before presenting to the user. Report the results as a pass/fail checklist.

### Factual accuracy
- [ ] All claims match actual profile (CLAUDE.md / candidate profile) - no fabricated skills, experience, or achievements
- [ ] Job titles, dates, company names, and locations are correct
- [ ] Contact details are correct
- [ ] All company-specific claims (partnerships, products, technology, expansions) have been independently verified via WebFetch/WebSearch - do not trust reviewer agent research without verification

### Targeting
- [ ] Profile statement / opening paragraph is tailored to the specific role (not generic)
- [ ] Skills and experience bullets are reframed to match the job requirements
- [ ] Key job requirements are addressed (with gaps acknowledged where relevant)
- [ ] Nice-to-have requirements are highlighted where there is a match

### Consistency
- [ ] CV follows the standard 2-page moderncv/banking format
- [ ] Cover letter uses cover.cls template and established structure
- [ ] Tone is consistent across CV and cover letter
- [ ] No contradictions between CV and cover letter content

### Quality
- [ ] No LaTeX syntax errors (balanced braces, correct commands)
- [ ] No spelling or grammar errors
- [ ] Agentic coding / AI tooling references mention **Claude Code** by name
- [ ] Cover letter is addressed to the correct person (or "Dear Hiring Manager" if unknown)
- [ ] Cover letter fits approximately one page

### Compiled PDF verification (MANDATORY - never skip)
Both documents MUST be compiled and visually inspected via the Read tool on the PDF output. "Looks fine in the .tex" is not acceptable - LaTeX page-break decisions are unpredictable. Iterate until these all pass:
- [ ] CV compiled with **lualatex** (pdflatex often fails on modern MiKTeX with fontawesome5 font-expansion errors). Cover letter compiled with **xelatex** (cover.cls requires fontspec).
- [ ] **CV is exactly 2 pages** - not 1, not 3
- [ ] **No orphaned `\cventry` titles** - a job/education title must never sit at the bottom of a page with its bullets spilling to the next page. Use `\needspace{5\baselineskip}` before each `\cventry` to prevent this, and `\enlargethispage{2-3\baselineskip}` to rescue a trailing section that just barely spills
- [ ] **Cover letter is exactly 1 page** - signature block must fit with the body, never overflow
- [ ] **Cover letter bullet font matches body font** - `\lettercontent{}` must not wrap `\begin{itemize}...\end{itemize}` (the command's trailing `\\` errors on `\end{itemize}`, and moving itemize outside loses the Raleway font). Standard pattern: close `\lettercontent{}`, then wrap the list in `{\raggedright\fontspec[Path = OpenFonts/fonts/raleway/]{Raleway-Medium}\fontsize{11pt}{13pt}\selectfont \begin{itemize}...\end{itemize}\par}`
