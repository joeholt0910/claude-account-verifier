---
description: Verify a batch of B2B accounts using parallel subagents, optionally cross-referenced against a customer list
argument-hint: [csv path OR pasted list of companies]
---

# Batch Account Verification Orchestrator

You are the orchestrator for a batch account verification. Input:

**$ARGUMENTS**

Your job: parse the input, ask about a customer list, dispatch parallel `verifier` subagents, apply customer matching, then write a CSV and in-chat summary. You do not verify companies yourself. The subagents do that.

## Step 1: Parse input

- **File path** (ends in `.csv`, starts with `/` or `~`, or is clearly a path): read the file. First column is company names. Skip header row if present.
- **Pasted list**: one company per line. Ignore blank lines. Strip numbering, bullets, whitespace.
- **Ambiguous or empty**: ask the user to clarify.

Deduplicate case-insensitively, preserving first occurrence's capitalization. Report the parsed count and dedupe count.

## Step 2: Prompt for customer list

After parsing the input and before dispatching subagents, ask the user:

> "Do you want to cross-reference these accounts against a customer list? If yes, provide a file path to a Salesforce Accounts report export (CSV) or paste the data. If no, reply 'skip' and I'll proceed without customer matching."

Wait for a response before proceeding.

- **File path provided**: read the file. Parse the CSV. Detect columns per the `customer-match` skill.
- **Pasted CSV/list provided**: parse the pasted content the same way.
- **"skip" or equivalent**: proceed without customer matching. Omit customer columns from the final CSV.

Once a customer list is loaded, report back: how many customer rows were loaded, which columns were detected (account name, domain, type, parent), and how many were classified as actual customers (vs. prospects filtered out).

## Step 3: Size check

- **≤ 5 companies**: skip subagent dispatch; verify inline using the `account-verification` skill directly.
- **6–20 companies**: proceed with subagent dispatch.
- **21–50 companies**: tell the user this will take meaningful time and tokens before proceeding.
- **51+ companies**: stop and ask the user to confirm. For 200+, recommend splitting into chunks of 100.

## Step 4: Check the knowledge base first (both inline and subagent paths)

Regardless of whether you're verifying inline (≤ 5) or dispatching subagents (6+), the knowledge base at `knowledge/reference.md` must be consulted first for each company.

- If a company appears in `knowledge/reference.md`, use that data with source "internal reference" and confidence H.
- If reference data is more than 12 months old, supplement with a web check on the most volatile fields (headcount), but keep the reference entry as the primary source. Note any deltas.
- Only fall back to full web research if the company is not in the reference.

## Step 5: Dispatch parallel verifier subagents (for 6+ company batches)

1. Chunk the deduplicated list into groups of up to 10 companies each
2. Spawn one `verifier` subagent per chunk, in parallel
3. Pass each subagent its chunk as a JSON array of company names
4. Wait for all subagents to complete and collect their JSON array outputs

## Step 6: Validate subagent output

For each returned row:
- Verify it has all expected verification fields
- If a subagent returned prose or malformed JSON, mark that chunk's companies with status `AGENT_ERROR` and include them in the output anyway
- Never silently drop rows

## Step 7: Apply customer matching

If a customer list was loaded in Step 2:

For each verified row, apply the `customer-match` skill to determine `customer_status` and `match_evidence`. Add these as two additional fields on each row.

**CRITICAL: The customer-match skill enforces strict first-match-wins ordering.** When applying the skill, evaluate rules 1-6 in order for each row and STOP at the first match. Do not override the skill's output. If a row matches Rule 1 (direct customer), it is "Customer" — even if the row also happens to be a parent of other customers. Never substitute a "more interesting" relational label for a direct match.

If no customer list was loaded, skip this step and do not include customer_status/match_evidence in the output.

## Step 8: Write CSV via Python (required — do NOT write CSV inline)

**Do not write CSV content inline using the Write tool directly.** Inline CSV writing has failed in the past with header corruption and RFC 4180 quoting bugs when notes fields contain commas and quotes. Instead, use Python with the `csv` module, which handles quoting correctly and mechanically.

Generate the filename using the current date and time:
```
verification-output-YYYY-MM-DD-HHMM.csv
```

For example: `verification-output-2026-04-22-0945.csv`. **Never overwrite an existing file.** If the exact timestamp collides (rare), append `-2`, `-3`, etc.

Use a Bash tool invocation with a Python heredoc that does the CSV write. Example structure:

```bash
python3 << 'PYEOF'
import csv
import os

# Build rows as a list of dicts
rows = [
    {
        "company_name": "Microsoft Corporation",
        "headcount": 228000,
        # ... all other fields ...
    },
    # ... more rows
]

# Columns (include customer fields ONLY if customer list was loaded)
columns = [
    "company_name", "headcount", "headcount_as_of", "headcount_source",
    "hq_city", "hq_region", "hq_source",
    "entity_classification", "parent_company",
    "confidence_headcount", "confidence_hq", "confidence_classification",
    "status",
    # "customer_status", "match_evidence",  # uncomment if customer list loaded
    "notes"
]

filename = "verification-output-2026-04-22-0945.csv"  # use actual timestamp
filepath = os.path.abspath(filename)

with open(filepath, "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=columns, quoting=csv.QUOTE_MINIMAL)
    writer.writeheader()
    for row in rows:
        # Ensure all columns exist in row, fill missing with ""
        writer.writerow({c: row.get(c, "") for c in columns})

print(f"Wrote {len(rows)} rows to {filepath}")
PYEOF
```

Adapt the structure to the actual rows and whether customer columns are included. `csv.QUOTE_MINIMAL` handles all quoting per RFC 4180 automatically — you do not need to hand-quote fields.

## Step 9: Validate the CSV after writing

Immediately after writing, read back the first 2 lines of the file and verify:

1. The header line contains all expected column names in the correct order, separated by commas
2. The first data row has the same number of comma-separated fields as the header (accounting for quoted fields that may contain commas)

If validation fails, report the error to the user and do not declare success. Attempt to rewrite once; if it fails again, report the file contents and stop.

## Step 10: In-chat summary

Produce:

1. **Headline counts**: e.g., "Verified 47 of 52 companies: 42 VERIFIED, 5 PARTIAL, 3 AMBIGUOUS, 2 NOT_FOUND"

2. **If customer list was loaded, customer breakdown**: e.g., "Customer matches: 8 Customer, 3 Customer — Dependent Child, 2 Customer Parent of Customer, 4 Likely Customer — Independent Child, 2 Sibling of Customer, 33 Not a Customer"

3. **Markdown table** with columns:
   - If customer list loaded: Company | Headcount | HQ | Classification | Customer Status | Status
   - Otherwise: Company | Headcount | HQ | Classification | Status

4. **Exceptions list**: brief bullets for non-VERIFIED rows

5. **Customer opportunities section (if list loaded)**: highlight the most interesting customer-relationship findings — especially "Customer Parent of Customer" (expansion opportunities) and "Likely Customer — Independent Child" rows (warm-intro candidates). Keep this short; it's a signal-flagging step, not a strategy memo.

6. **Reference candidates**: list EVERY row that meets all of these criteria:
   - `status` is `VERIFIED`
   - All three confidence fields are `H`
   - The company came from fresh research (NOT from `knowledge/reference.md`)

   Do not second-guess whether they're "worth caching" — if they meet the criteria, list them. The user decides what actually goes into reference.md. Offer to append them.

## Rules

- Never fabricate data. `UNVERIFIED` and "Not a Customer" are always valid.
- Always produce both outputs. The CSV is always written with a timestamped filename.
- **Always write the CSV via Python with `csv.DictWriter`, not inline.** Inline writing has produced corrupted output in the past.
- Always validate the CSV after writing (Step 9) before declaring success.
- If a subagent fails, affected companies still appear with status `AGENT_ERROR`.
- Customer matching is never silently skipped if a list was loaded — every row gets a customer_status.
- **Apply customer-match rules in strict order (1→6) and stop at first match.** Never substitute a relational label (Rule 3+) for a direct match (Rule 1).
- If the user said "skip" to the customer list, do not prompt again later in the same run.
- Never overwrite an existing CSV file. Each run produces a new timestamped file.
