# Interview Preparation Guide

<!-- SETUP: STAR examples are personalized by running /setup based on your actual experience -->

## STAR Format

Structure answers as: **Situation** (context), **Task** (your responsibility), **Action** (what you did), **Result** (outcome).

Keep answers to 1-2 minutes. Be specific. End with what you learned or would do differently.

## Ready-Made STAR Examples

### 1. AnaCredit Migration Testing (Data quality automation)
**S:** At Deutsche Bundesbank's AnaCredit unit, a data migration risked introducing silent discrepancies in business logic (row counts, referential integrity, aggregate values) between old and new systems.
**T:** Build automated tests to catch any deviation before it reached reporting banks or downstream processes.
**A:** Developed automated migration tests using Pandas DataFrames covering row-count checks, referential integrity, and aggregate consistency. Layered on automated error detection for duplicate IDs and value-range violations, wired to automatically create Jira tickets and notify reporting banks.
**R:** Significantly reduced response time to data quality issues, catching problems automatically instead of relying on manual review.
**Use for:** "Tell me about a time you improved a process", "Describe your experience with data quality/testing", "How do you approach automation?"

### 2. Lattice-Based Cryptography Research (Independent research & rigor)
**S:** As part of a research project at Tecnológico de Monterrey under Prof. A. F. De Abiega L'Eglisse, the team needed to understand the practical security and performance trade-offs of lattice-based cryptographic schemes like LWE and NTRU.
**T:** Implement the algorithms and analyze their complexity and security parameters across different lattice dimensions.
**A:** Implemented LWE and NTRU in Python, ran complexity analysis, and studied runtime behavior as lattice dimensions changed, connecting the results to security parameter selection.
**R:** Contributed analysis that fed into a scientific publication on the topic.
**Use for:** "Tell me about independent/research work", "Describe a technically challenging project", "Why are you interested in security/cryptography?"

### 3. Healthcare Predictive Modeling (Applied ML on real-world data)
**S:** Mexico's national health system had over 25 years of data that hadn't been systematically mined for patient risk patterns or resource planning.
**T:** Build models to identify patient risk profiles and resource bottlenecks from this historical data.
**A:** Built predictive models (Random Forest, Neural Networks) to forecast critical outcomes, used K-Means clustering to segment high-risk patient groups, and ran exploratory time-series and regional analyses to find seasonal patterns.
**R:** Surfaced risk profiles and seasonal resource bottlenecks that could inform planning decisions.
**Use for:** "Tell me about a machine learning project end-to-end", "How do you approach an open-ended data problem?"

### 4. Personal Banking Data Pipeline (Ownership & auditability)
**S:** Wanted to demonstrate the ability to build a production-style, audit-ready data pipeline independent of any course or employer assignment.
**T:** Design and build an end-to-end pipeline for simulated bank transaction data that could pass an audit.
**A:** Simulated realistic transaction data for 150 accounts, loaded it into a normalized SQLite database with foreign-key integrity checks, and built staged data cleaning (duplicates, nulls, date formats, fuzzy-matched categorical errors) with a fully auditable log recording every change (table, column, old/new value, action, timestamp). Added SQL triggers for automatic balance maintenance and built 5 quality visualizations plus a structured exception report.
**R:** Produced a fully self-contained, auditable pipeline showcased on GitHub, demonstrating initiative beyond coursework or internship requirements.
**Use for:** "Tell me about a self-directed project", "How do you ensure data quality?", "Describe your experience with SQL/databases"

## Common Tough Questions

### "Why did you leave [previous company]?"
> [PREPARE YOUR ANSWER - be honest, forward-looking, no negativity about former employer]

### "You don't have [specific skill/experience]."
> [PREPARE YOUR ANSWER - acknowledge the gap, bridge to adjacent experience, show willingness to learn]

### "Where do you see yourself in 5 years?"
> [PREPARE YOUR ANSWER - show ambition aligned with the role's growth path]

### "What's your biggest weakness?"
> [PREPARE YOUR ANSWER - genuine weakness with concrete mitigation strategy]

### "Why this company specifically?"
> Customize per company. Must reference: specific projects, company values, market position, or team structure. Never give a generic answer.

## Questions You Should Ask Interviewers

### About the Role
- "What does a typical week look like in this role?"
- "What would success look like in the first 6 months?"
- "What's the biggest challenge the team is facing right now?"

### About the Team
- "How big is the team, and how do you divide work?"
- "What does the development/project lifecycle look like, from idea to production?"
- "How do you onboard new team members?"

### About Tech & Growth
- "What's your current tech stack for [relevant area]?"
- "Is there room to grow into more architectural or strategic decisions?"
- "How does the team stay current with new tools and methods?"

### About Culture (use these to prevent disappointment)
- "How would you describe the team culture?"
- "What does professional development look like here?"
- "Is there flexibility for remote/hybrid work?"
- "What's the balance between development/new projects and maintenance work?"
- "How would you describe the leadership style in this team?"
- "What do people who thrive here have in common?"

## Phone/Video Interview Tips
- Have STAR examples written out (use this file)
- Keep a glass of water nearby
- Smile when speaking (it changes your tone)
- Ask for clarification if a question is vague
- It's OK to take 5 seconds to think before answering
- End with: "Is there anything else you'd like to know about my background?"

## After the Application (Best Practice)

### Follow-Up Etiquette
- **Don't call to "stand out"** or to learn more about the role post-submission - this risks a negative impression
- If the employer specified a timeline, respect it and wait
- If no timeline was given and significant time has passed (2+ weeks), a brief call to ask about status is acceptable
- If you have genuinely new, relevant information to share, a short follow-up is fine

### Thank-You Notes
- When you receive any update (interview invitation, rejection, or status update), send a brief thank-you message
- Express appreciation for their time and the process
- Keep it short (2-3 sentences)

## Roleplay Guidelines
When the user asks for interview practice:
1. Ask which role/company to simulate
2. Start with easy warm-up questions ("Tell me about yourself")
3. Progress to role-specific technical questions
4. Include 1-2 behavioral questions using the competencies from the job posting
5. End with a tough question or curveball
6. After each answer, give brief feedback: what worked, what to sharpen
7. Suggest which STAR example would work best for each question
