---
name: account-verification
description: Use when verifying B2B enterprise accounts for employee headcount, HQ location, and entity classification (Parent, Standalone, Independent Child, Dependent Child). Applies to sales operations research, account data quality projects, territory planning, Salesforce account hygiene, and any task requiring authoritative company firmographic data with source citations and confidence levels.
---

# Account Verification Methodology

This skill defines the research methodology and output format for verifying B2B enterprise accounts. It is used by the `/verify-account` command and should also activate automatically for any account verification request.

## Source Hierarchy

Use sources in this order. Do not skip tiers unless a higher tier is genuinely unavailable, and always note which tier the data came from.

### Tier 1: Primary Filings (highest confidence)
- **SEC filings** — 10-K, 10-Q, DEF 14A, S-1. Best for public US companies. Headcount is typically in Item 1 or the Human Capital section.
- **Official foreign equivalents** — UK annual reports filed at Companies House, Canadian SEDAR filings, etc.

### Tier 2: Official Company Publications
- **Annual reports** and **investor relations pages** — for public non-US companies and large private companies that publish them
- **Official company "About" or "Careers" pages** — for HQ and sometimes headcount
- **Official press releases** announcing headcount milestones

### Tier 3: Reputable Platforms
- **LinkedIn company page** — useful for directional headcount (employees on platform), but note this is an undercount of total employees
- **Bloomberg, Reuters, Financial Times** — for entity structure, M&A history, and HQ
- **Major business publications** reporting on the company

### Tier 4: Aggregators (last resort)
- **ZoomInfo, PitchBook, Crunchbase, Owler** — flag any data from these as Low confidence and note the aggregator by name
- **Never cite Wikipedia as a primary source.** Wikipedia is acceptable only as a starting point to find higher-tier sources.

## Entity Classification

Use these definitions:

- **Parent** — Has operating subsidiaries; is not itself owned by another operating company. Example: Microsoft Corporation.
- **Standalone** — No subsidiaries, no parent. Operates as a single legal entity. Example: a mid-market SaaS company pre-acquisition.
- **Independent Child** — Owned by a parent but operates with meaningful autonomy: separate P&L, distinct GTM motion, often a distinct brand. Example: LinkedIn under Microsoft.
- **Dependent Child** — Owned by a parent and operationally integrated: shared systems, consolidated GTM, unified brand or sub-brand. Example: a fully integrated acquired product line.

When classification is genuinely ambiguous (e.g., a recently acquired company mid-integration), default to Independent Child and flag the ambiguity in the Notes field.

## Headcount Guidance

- Prefer **global total employee count**. If only US or regional figures are available, use them and explicitly label as such.
- Always include an **as-of date**. "Approximately 50,000 employees" is less useful than "49,700 as of December 31, 2024 (10-K)."
- Exclude contractors unless the source explicitly bundles them. Note if the source does.
- LinkedIn headcount is "employees with LinkedIn profiles" — it is directional, not authoritative. Treat as Medium confidence at best.

## Output Format

Return this exact structure. Use Markdown.

**Company:** [Full legal name]
**Headcount:** [Number] (as of [Date], source: [Source])
**HQ:** [City, State/Country] (source: [Source])
**Entity Classification:** [Parent / Standalone / Independent Child / Dependent Child]
**Parent (if applicable):** [Name, or "N/A"]
**Confidence:** Headcount [H/M/L] | HQ [H/M/L] | Classification [H/M/L]
**Notes:** [Judgment calls, conflicts between sources, ambiguities, or "None"]

## Rules

1. **Never guess.** "Unknown" or "Needs manual review" is always a valid answer. Fabrication is a severe error.
2. **Cite sources inline.** Every data point needs a source named in parentheses.
3. **Flag conflicts.** If two Tier 1 or Tier 2 sources disagree on headcount, surface both, choose one, and explain why.
4. **Disambiguate when needed.** If the company name matches multiple entities (e.g., "Acme Corp" could be several companies), ask before researching.
5. **Confidence is per-field, not overall.** A company might have High confidence on HQ (from their website) and Low confidence on headcount (only aggregator data available).

## Example Output

**Company:** Ford Motor Company
**Headcount:** 171,000 (as of December 31, 2024, source: Ford 2024 10-K)
**HQ:** Dearborn, Michigan, USA (source: Ford 10-K, ford.com)
**Entity Classification:** Parent
**Parent (if applicable):** N/A
**Confidence:** Headcount H | HQ H | Classification H
**Notes:** Ford has numerous operating subsidiaries (Ford Credit, Lincoln, etc.), confirming Parent classification. Global headcount includes hourly and salaried employees; excludes contractors per 10-K disclosure.
