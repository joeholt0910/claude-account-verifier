# Account Verification Reference Data

This file is a curated, manually-verified reference for accounts your team researches frequently. When the plugin verifies a company, it **checks this file first**. If the company appears here, the plugin uses this data and marks confidence as High with source "internal reference" — skipping web research entirely.

This is how the plugin gets fast: you build up your own institutional reference over time. After the plugin runs fresh research on a company and returns High confidence across all fields, it will offer to append that entry here. Accept when the data is stable (Fortune 500 parents) and skip when it's volatile (small companies, recent acquisitions).

If reference data is more than 12 months old, the plugin will supplement with a fresh web check on volatile fields (headcount), but keep the reference entry as primary. Any deltas get flagged in the output.

## Format

Each entry follows this structure:

```
### [Company Name]
- Headcount: [number] (as of [date], source: [source])
- HQ: [city, region]
- Entity Classification: [Parent / Standalone / Independent Child / Dependent Child]
- Parent: [name or N/A]
- Last verified: [YYYY-MM-DD]
- Notes: [anything worth preserving]
```

## Example Entries

These are starter entries to show the format. Replace or extend as you verify accounts relevant to your org.

### Ford Motor Company
- Headcount: 171,000 (as of 2024-12-31, source: Ford 2024 10-K)
- HQ: Dearborn, Michigan, USA
- Entity Classification: Parent
- Parent: N/A
- Last verified: 2026-04-22
- Notes: Operates Ford Credit, Lincoln, and other subsidiaries.

### Microsoft Corporation
- Headcount: 228,000 (as of 2024-06-30, source: Microsoft 2024 10-K)
- HQ: Redmond, Washington, USA
- Entity Classification: Parent
- Parent: N/A
- Last verified: 2026-04-22
- Notes: Owns LinkedIn, GitHub, Activision Blizzard as independent children.

### LinkedIn Corporation
- Headcount: ~21,000 (as of 2024, source: LinkedIn company reports)
- HQ: Sunnyvale, California, USA
- Entity Classification: Independent Child
- Parent: Microsoft Corporation
- Last verified: 2026-04-22
- Notes: Acquired by Microsoft 2016, operates with distinct brand and P&L.

## Your Frequently-Referenced Accounts

(Populate this section with accounts your team verifies often. Every High-confidence fresh verification the plugin runs can be appended here to build up your internal reference over time.)

## Regional / International Accounts

(Populate this section with verified accounts outside your primary region. International entity structures are often complex — worth preserving once researched.)
