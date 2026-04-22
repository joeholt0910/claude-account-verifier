# verify-account

A [Claude Code](https://code.claude.com) plugin for batch B2B account verification. Built for sales operations teams dealing with Salesforce data quality problems — territory planning, ABM prioritization, account hygiene, and large-scale verification projects.

Given a list of companies, it returns structured firmographic data (headcount, HQ, entity classification) with confidence levels, source citations, and — if you provide a customer list — flags each account against your existing customer base with inheritance for parent/child relationships.

## What it does

- Verifies accounts against a strict source hierarchy: SEC filings and foreign-equivalent regulatory filings first, then official annual reports and company sites, then reputable platforms, and aggregators only as a flagged last resort.
- Classifies each account as Parent, Standalone, Independent Child, or Dependent Child with parent-company linkage where applicable.
- Cross-references against a Salesforce Accounts report export and labels each account across six tiers: Customer (direct match), Customer — Dependent Child, Customer Parent of Customer, Likely Customer — Independent Child, Sibling of Customer, or Not a Customer. First match wins, with auditable match evidence per row.
- Filters Prospect and Former Customer rows out of matching automatically.
- Dispatches parallel subagents for batches of six or more so research on independent accounts happens concurrently, not sequentially.
- Maintains a self-populating knowledge base: accounts verified at High confidence across all fields are offered as candidates to cache, so repeat lookups hit memory instead of running fresh research.
- Writes a timestamped CSV (RFC 4180-compliant) alongside an in-chat markdown summary with exceptions and expansion-opportunity flags.

## Installation

Requires Claude Code 2.x.

```bash
git clone https://github.com/joeholt0910/claude-account-verifier.git ~/claude-account-verifier
```

Then in Claude Code:

```
/plugin marketplace add ~/claude-account-verifier
/plugin install verify-account@claude-account-verifier
/reload-plugins
```

You should see: `Reloaded: 1 plugin · 2 commands · 2 skills · 1 agent`.

Confirm registration by typing `/` in the prompt — `verify-account:batch` and `verify-account:verify-account` should appear in autocomplete.

## Quick start

### Verify a single account

```
/verify-account:verify-account Ford Motor Company
```

Returns a structured block with headcount, HQ, classification, confidence per field, and sources.

### Verify a batch

```
/verify-account:batch
Ford Motor Company
Microsoft Corporation
Toyota Motor Corporation
```

Or point it at a CSV:

```
/verify-account:batch ~/accounts-to-verify.csv
```

The plugin prompts for a customer list before dispatching verification. Reply with a file path, paste CSV content, or type `skip` to proceed without customer matching.

### Cross-reference against customers

Export your customer accounts from Salesforce as a report with these columns (names are detected flexibly):

| Column | Example header names accepted |
|---|---|
| Account Name | Account Name, Name, Account, Company |
| Website | Website, Domain, URL |
| Type | Type, Account Type, Status |
| Parent | Parent Account, Parent, Ultimate Parent |

When the plugin prompts you for a customer list, provide the path. It will detect your columns, filter out Prospects, and cross-reference every verified account against the customer pool.

## Output

Two outputs per run:

**CSV file** — written to your working directory with a timestamped filename (`verification-output-YYYY-MM-DD-HHMM.csv`). Never overwrites. Columns vary based on whether a customer list was loaded.

**In-chat summary** — headline counts, markdown table, exceptions list, customer-opportunity flags (expansion candidates, warm-intro signals), and reference-candidate offers.

Example summary when a customer list is loaded:

```
Verified 52 of 52 companies: 47 VERIFIED, 3 PARTIAL, 2 AMBIGUOUS

Customer matches: 8 Customer, 3 Customer — Dependent Child,
2 Customer Parent of Customer, 4 Likely Customer — Independent Child,
2 Sibling of Customer, 33 Not a Customer
```

## Customizing for your org

### Build up your reference base

The file `knowledge/reference.md` is where the plugin looks first before doing web research. Out of the box it contains a few example Fortune 500 entries. After each batch run, the plugin will offer to append High-confidence verifications to this file. Accept for stable accounts (large public parents) and skip for volatile ones (recent acquisitions, startups). Over time, your reference file becomes a genuine institutional asset — your team's verified truth, not aggregator data.

You can also manually pre-populate it for accounts your team researches often.

### Adjust source hierarchy or entity classification rules

The core methodology lives in `skills/account-verification/SKILL.md`. Edit this file to tighten source requirements, add industry-specific classification rules, or adjust confidence thresholds. After editing, uninstall and reinstall the plugin so Claude Code picks up the change from the cached copy:

```
/plugin uninstall verify-account@claude-account-verifier
/plugin install verify-account@claude-account-verifier
/reload-plugins
```

### Adjust customer-matching rules

The six-tier matching logic lives in `skills/customer-match/SKILL.md`. Edit the rules, labels, or fuzzy-matching thresholds to suit your org's customer-tier taxonomy.

## Scale guidance

| Batch size | Behavior |
|---|---|
| 1–5 | Verified inline (no subagent overhead) |
| 6–20 | One or two parallel subagents |
| 21–50 | Multiple parallel subagents; plugin confirms before proceeding |
| 51+ | Plugin stops and asks for confirmation |
| 200+ | Split into chunks of 100 for best results |

A typical 15-company batch runs in 2–3 minutes with no manual intervention.

## Troubleshooting

**"Unknown command: /verify-account:batch"** — the plugin installed but commands didn't register. Try `/reload-plugins`. If that doesn't work, uninstall and reinstall.

**CSV not being written** — check whether Claude Code prompted you to approve a Python command starting with `python3 << 'PYEOF'`. The plugin writes CSV via Python to avoid quoting bugs; you need to approve the shell invocation.

**Subagents returning AGENT_ERROR** — one or more subagents failed to return valid JSON, usually from context pressure. Break the batch into smaller chunks (10 companies or fewer) and rerun.

**Commands aren't showing up in autocomplete** — confirm the plugin is installed via `/plugin` → Installed tab. If it's listed there but commands aren't appearing, reinstall.

## Design principles

- **No fabrication.** "UNVERIFIED" and "Not a Customer" are always valid values. Fabricated data is worse than missing data.
- **Source everything.** Every claim cites a specific document. A verification someone can't audit isn't useful.
- **Confidence is per-field.** An account can have High confidence on HQ and Medium on headcount. Overall-confidence summaries hide the signal.
- **First match wins for customer labels.** A company that's both a direct customer and a parent-of-customers is simply a Customer. Relational labels are only applied when the simpler direct label doesn't fit.
- **Surface the exceptions.** Any row that required judgment, had conflicting sources, or hit a classification edge case gets flagged for human review.

## License

MIT. See [LICENSE](./LICENSE) for details.

## Author

Joe Holt. Originally developed for B2B sales operations workflows. Feedback and issues welcome via GitHub.
