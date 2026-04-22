---
description: Verify a single B2B account (headcount, HQ, entity classification), optionally cross-referenced against a customer list
argument-hint: [company name]
---

# Verify Account

You are verifying the following company: **$ARGUMENTS**

## Step 1: Verify

Use the `account-verification` skill to research and return:

- Employee headcount (global, with as-of date and source)
- HQ location (city, state/country, with source)
- Entity classification (Parent / Standalone / Independent Child / Dependent Child)
- Parent company, if applicable
- Confidence level per field (H / M / L) with source cited

Always check `knowledge/reference.md` first before web research. If the company is in the reference, use that data.

## Step 2: Offer customer matching

After the verification, ask the user:

> "Do you want to cross-reference this account against a customer list? Provide a file path to a Salesforce Accounts report export (CSV) or paste the data. Reply 'skip' to skip."

- If a list is provided, parse it and apply the `customer-match` skill. Return `customer_status` and `match_evidence`.
- If 'skip', proceed without.
- If the user has already loaded a customer list earlier in this session, use it without asking.

## Step 3: Return the verification

Format per the `account-verification` skill's output template. If a customer match was performed, add:

**Customer Status:** [status]
**Match Evidence:** [evidence]

## Rules

- Never fabricate data.
- Never guess at matches. If uncertain, use "Not a Customer" and explain uncertainty in match_evidence.
- Disambiguate before proceeding if the company name is ambiguous.
