---
name: customer-match
description: Use when matching a verified B2B account against an existing customer list (typically a Salesforce report export) to determine whether the account is a current customer, a child of a customer, a parent of a customer, or a sibling of a customer. Returns a customer_status label and evidence for the match. Always activates when a customer list has been provided in the session.
---

# Customer Match Methodology

This skill matches verified accounts against a customer list (usually a Salesforce Accounts report) and assigns a `customer_status` label. It is used by the `/verify-account` commands when a customer list is available.

## CRITICAL: Evaluation Order

**You MUST evaluate rules in strict numerical order (1 → 6) and STOP at the first match.**

This is not a preference. It is the core rule of this skill. A common failure mode is to find a "more interesting" relational match (like "this is a parent of other customers!") and apply that label instead of the simpler direct match. Do not do this. If Rule 1 matches, you stop at Rule 1. You never look at Rules 2-6.

Before assigning a label, ask yourself: "Did I check Rule 1 first? Did I check Rule 2 second?" If you find yourself at Rule 3 without having conclusively ruled out Rules 1 and 2, start over.

## Input

Two things must be in scope:

1. **The verified account** — with `company_name`, `hq_city`, `hq_region`, `entity_classification`, and `parent_company`
2. **The customer list** — a parsed Salesforce export with at minimum: account name, and ideally website/domain, account type (Customer vs. Prospect vs. other), and parent account

## Column detection

Salesforce reports vary across orgs. Detect columns flexibly — look for:

- **Account name**: "Account Name", "Name", "Account", "Company"
- **Domain/website**: "Website", "Domain", "URL", "Company Website"
- **Customer type**: "Type", "Account Type", "Status", "Customer Status"
- **Parent**: "Parent Account", "Parent", "Ultimate Parent"

Only treat a row as a "customer" if the account type column indicates it. Common customer values: "Customer", "Customer - Active", "Active Customer", "Current Customer". If there is no type column at all, treat every row as a customer (the user uploaded a filtered customer-only list).

## Domain derivation

For each customer row with a website, extract the root domain:
- `https://www.acme.com/about` → `acme.com`
- `acme.co.uk` → `acme.co.uk`
- Strip protocol, `www.`, paths, query strings
- Lowercase everything

For each verified account, derive the likely domain if possible:
- Use known official domains when you have them
- If you must guess, don't — leave the domain match step out rather than false-matching

## Matching rules (MUST be applied in order, first match wins)

### Rule 1 — Direct match → Customer

Check this FIRST for every verified account. Do not skip. Do not combine with other rules.

- Verified account's domain exactly matches a customer row's domain, OR
- Verified account's name fuzzy-matches a customer row's name at ≥ 95% similarity

Use reasonable fuzzy-matching: normalize case, strip legal suffixes ("Inc", "Corp", "Corporation", "Ltd", "LLC", "GmbH", "AG", "S.A.", "plc") before comparing. "Ford Motor Company" and "Ford Motor Co." should match. "Apple Inc." and "Apple Hospitality REIT" should not.

**If this rule matches, STOP. Output `customer_status: "Customer"`.** Do not evaluate rules 2-6, even if this account also happens to be a parent of other customers, a child of a customer, etc.

**Output**: `customer_status: "Customer"`, `match_evidence: "[exact domain match | fuzzy name match N%] against '[customer list name]'"`

### Rule 2 — Customer — Dependent Child

Only reached if Rule 1 did NOT match.

- Verified account's `entity_classification` is "Dependent Child", AND
- Verified account's `parent_company` matches a customer row (by the same domain/fuzzy name rules above)

**If this rule matches, STOP.**

**Output**: `customer_status: "Customer — Dependent Child"`, `match_evidence: "Parent '[parent]' matches customer list"`

### Rule 3 — Customer Parent of Customer

Only reached if Rules 1 and 2 did NOT match.

- Any customer row's parent (from the Parent Account column) matches the verified account by name/domain

**If this rule matches, STOP.**

**Output**: `customer_status: "Customer Parent of Customer"`, `match_evidence: "Customer '[customer name]' has this account as its parent"`

### Rule 4 — Likely Customer — Independent Child

Only reached if Rules 1-3 did NOT match.

- Verified account's `entity_classification` is "Independent Child", AND
- Verified account's `parent_company` matches a customer row

**If this rule matches, STOP.**

**Output**: `customer_status: "Likely Customer — Independent Child"`, `match_evidence: "Parent '[parent]' is a customer; separate P&L likely means separate GTM"`

### Rule 5 — Sibling of Customer

Only reached if Rules 1-4 did NOT match.

- Verified account's `parent_company` matches the Parent Account of any customer row, AND
- The verified account itself does not match rules 1–4

**Output**: `customer_status: "Sibling of Customer"`, `match_evidence: "Shares parent '[parent]' with customer '[sibling customer]'"`

### Rule 6 — Not a Customer

Only reached if Rules 1-5 did NOT match.

**Output**: `customer_status: "Not a Customer"`, `match_evidence: ""`

## Worked Examples

These examples clarify the "first match wins" behavior in edge cases.

### Example A: Direct customer that's also a parent

Verified account: **Microsoft Corporation** (Parent entity, no parent company)

Customer list contains:
- Microsoft Corporation (Customer, no parent)
- LinkedIn Corporation (Customer, parent = Microsoft Corporation)
- GitHub Inc (Customer, parent = Microsoft Corporation)

Microsoft matches Rule 1 (direct name match on "Microsoft Corporation"). **STOP.** Output: `Customer`.

Do NOT apply Rule 3 ("Customer Parent of Customer") even though Microsoft is also the parent account for LinkedIn and GitHub. Rule 1 fired; we stop at Rule 1.

### Example B: Child of a customer, where the child is NOT itself a customer

Verified account: **Xbox Game Studios** (Dependent Child, parent = Microsoft Corporation)

Customer list contains:
- Microsoft Corporation (Customer)

Xbox Game Studios does not match Rule 1 (no row for "Xbox Game Studios" in the list). Check Rule 2: is this a Dependent Child whose parent is a customer? Yes. **STOP.** Output: `Customer — Dependent Child`.

### Example C: Prospect that shares a name

Verified account: **Ford Motor Company**

Customer list contains:
- Ford Motor Company (Prospect, not Customer)

Ford Motor Company is PRESENT in the customer list, but its Account Type is Prospect. Per the "Account type matters" rule below, Prospect rows are excluded from matching. Rule 1 does NOT match (no Customer row with this name). Rules 2-5 do not apply. Output: `Not a Customer`.

Include a note in `match_evidence`: "Name appears in customer list as Prospect — excluded from matching."

## Rules

1. **First match wins — STRICTLY.** Do not combine labels. A direct customer whose parent is also a customer is simply "Customer." See Example A.
2. **Never fabricate matches.** If you're uncertain, default to "Not a Customer" and note the uncertainty in match_evidence.
3. **Account type matters.** A row whose type is "Prospect" or "Former Customer" is not a customer — do not match against those rows unless the user's list has no type column at all. See Example C.
4. **Case and legal-suffix normalization are required.** Without them, half of real matches are missed.
5. **When in doubt about the parent relationship, demote the label.** If you're not certain the verified account's parent is actually the customer in the list (vs. a similarly-named different entity), use "Not a Customer" and flag in match_evidence.
6. **Always produce `match_evidence` that names the specific rule that fired.** The evidence should let the user audit your decision. "Rule 1: direct name match on 'Microsoft Corporation' (Customer)" is auditable. "Direct match" alone is not.
