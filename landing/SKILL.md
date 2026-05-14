---
name: finance-billing
description: "Turn a monthly credit card or bank statement PDF into reimbursement-ready CSVs with markup, tax, bill-to blocks, and a self-verification pass. You define the buckets, the merchant rules, the markup %, the tax %, and the bill-to blocks once — the skill runs the recipe every month after. Generic, configurable version of a workflow that replaced ~15 days of monthly finance work for a small business. Triggers on monthly close, credit card statement processing, expense categorization, reimbursement invoicing, vendor bucketing, or bill-to-block generation."
---

# Finance Billing

A five-layer recipe for turning a credit card or bank statement PDF into reimbursement-ready CSVs — categorized, totaled, marked-up, taxed, bill-to blocks attached, and self-verified before it ever reaches you.

This skill exists because most monthly-close work isn't finance. It's typing. Once the decisions (categorization, exclusions, markup, tax, bill-to) are written down, the recipe runs the same way every month.

## How to use

1. Drop a statement PDF (or a path to one) into the chat.
2. Tell Claude: *"Process this statement with the finance-billing skill."*
3. On first run, the skill will ask you for the **config bundle** (buckets, merchant rules, markup %, tax %, bill-to blocks, exclusion list). It will save these to `finance-billing-config.yaml` in the working directory.
4. On every subsequent run, the skill loads the saved config and just asks you to confirm any edge cases.

## Layer 1 — Extract

From the statement PDF:

1. Parse with PyMuPDF (`fitz`). Open the PDF, iterate pages, extract text rows.
2. **Take every debit entry** (purchases, charges, fees).
3. **Drop every credit entry** (payments received, refunds, cashbacks, statement-credit reversals).
4. Capture, per row: transaction date, merchant name, foreign-currency amount (if any), home-currency amount.
5. Use the **statement's transaction date**, never a posting date or a date supplied by anyone downstream. If a third-party (CA, bookkeeper) sends a sheet later with different dates, the statement wins.
6. Apply the user-supplied **exclusion list** (Layer 1b). Drop any row whose merchant or pattern matches.
7. **Forex fee + tax on forex fee** stay in. Those are part of the cost.

### Layer 1b — Exclusion list (config)

Stored in `finance-billing-config.yaml` under `exclusions:`. Examples of what users typically put here (don't include any of these by default — ask the user):

```yaml
exclusions:
  - pattern: "refunded in same statement"
    note: "Items with offsetting credits"
  - merchant: "<your-own-infra-vendor>"
    note: "Paid by parent entity, not reimbursable"
  - merchant: "<annual-card-fee-pattern>"
    note: "Not a billable expense"
```

If a row matches an exclusion, drop it and log to a `_excluded.csv` so the user can audit what was removed.

## Layer 2 — Categorize

Each remaining row goes into exactly one **bucket**. Buckets are defined by the user at first run. There can be any number (2 is common; the article's author used 2; some businesses need 5).

Config shape:

```yaml
buckets:
  - name: "Marketing"
    merchants:
      - "<ad-platform-A>"
      - "<ad-platform-B>"
  - name: "Operations"
    merchants:
      - "<infra-vendor>"
      - "<saas-tool>"
  - name: "Personal"
    merchants:
      - "<personal-pattern>"
    billable: false
```

Categorization rule, in order:

1. If the merchant matches a `merchants` entry under a bucket, assign that bucket.
2. If no bucket matches, **ask the user**: *"Merchant `X` did not match any bucket. Which bucket? [list]. Or add a new one."* Append the answer to the config so it's auto-matched next month.
3. Rows in any bucket where `billable: false` are excluded from the totals math (Layer 3) but still appear in `_personal.csv` for the user's records.

Per the article: "If the merchant is on this short list of ad platforms, it's marketing. Otherwise it's operational." Keep the rule embarrassingly short. It works.

## Layer 3 — Math

For each billable bucket, compute and emit:

```
Reimbursement       = sum of row amounts in this bucket (home currency)
Markup              = Reimbursement × (markup_pct / 100)
Tax on markup       = Markup × (tax_pct / 100)
Grand Total         = Reimbursement + Markup + Tax on markup
```

Config:

```yaml
markup_pct: 8          # default; the article's author used 8
tax_pct: 18            # default; India GST on services. Set to 0 for non-tax jurisdictions.
home_currency: "INR"
```

Round every figure to **two decimal places** at the rupee/cent level. Verify the sum-of-rows equals the bucket subtotal before applying markup — if not, halt and surface the discrepancy.

## Layer 4 — Bill-to blocks

Each billable bucket gets attached to one or more bill-to blocks. The user provides these once. The skill never asks the user to retype them again.

Config:

```yaml
bill_to_blocks:
  primary:
    legal_name: "<Entity Legal Name>"
    address_line_1: "<Building / Flat>"
    address_line_2: "<Street>"
    locality: "<Locality>"
    city: "<City>"
    state: "<State>"
    pin: "<PIN / ZIP>"
    tax_id: "<GSTIN / VAT / EIN>"
    tax_id_label: "GSTIN"   # or "VAT", "EIN", etc.
  alternate:
    # same shape; used if the user opts to split billing
    legal_name: "..."
```

The output CSV for each bucket includes the bill-to block as a footer block, formatted as plain text (one field per line), so it can be pasted into an invoice template as-is.

## Layer 5 — Self-verification (run BEFORE showing the user anything)

This is the layer most people skip and shouldn't. The skill runs every one of these checks against its own output. If any check fails, **do not present the result** — surface the failing check with the relevant rows and the math.

Checklist:

- [ ] **Debit count parity.** The number of rows in all output CSVs (billable + non-billable + excluded) equals the statement's total purchase count. If the statement prints a "Total Purchases" line, match against it.
- [ ] **Zero credits.** No row in any output CSV has a negative amount or a credit flag.
- [ ] **Exclusions logged.** Every excluded row is in `_excluded.csv` with the matching exclusion reason.
- [ ] **Bucket sum integrity.** For each bucket, sum-of-row-amounts equals the printed bucket subtotal, to the rupee/cent.
- [ ] **Markup arithmetic.** `Markup == Reimbursement × markup_pct / 100`, to the rupee/cent. (Recompute and compare; never trust the variable.)
- [ ] **Tax arithmetic.** `Tax == Markup × tax_pct / 100`, to the rupee/cent.
- [ ] **Grand total arithmetic.** `Grand Total == Reimbursement + Markup + Tax`, to the rupee/cent.
- [ ] **Bill-to completeness.** Every billable bucket has a fully-populated bill-to block — no empty fields, no `<placeholder>` strings.
- [ ] **Forex line items present.** If the statement contains any foreign-currency transactions, the corresponding forex-fee and tax-on-forex-fee rows are present and assigned to the same bucket as their parent transaction.
- [ ] **No row appears in two buckets.** Every row has exactly one bucket assignment.

If any check fails, the skill outputs:

```
VERIFICATION FAILED: <check name>
Failing rows / values:
<table>
Math:
<the recomputation that disagrees>
```

…and stops. No CSV is shipped until every check passes.

## Output

For a statement with N billable buckets, the skill produces:

```
<month>/
├── <bucket-1>.csv            # rows + footer block (totals, markup, tax, grand total, bill-to)
├── <bucket-2>.csv
├── _personal.csv              # non-billable rows, for user's records
├── _excluded.csv              # excluded rows with exclusion reasons
└── _verification.txt          # the five-layer self-check log
```

CSV column shape per bucket:

```
SN, Merchant, Foreign Currency Amount, Foreign Currency, Home Currency Amount, Date, Bill Status
```

Footer block (appended after the data rows):

```
TOTAL                  <Reimbursement>
Markup (<pct>%)        <Markup>
Tax on Markup (<pct>%) <Tax>
GRAND TOTAL            <Grand Total>

BILL TO:
<legal_name>
<address_line_1>
<address_line_2>
<locality>, <city>
<state> — <pin>
<tax_id_label>: <tax_id>
```

## First-run config wizard

If `finance-billing-config.yaml` is not present in the working directory, the skill walks the user through:

1. How many buckets? Name each.
2. For each bucket: is it billable? List a few sample merchants (more can be added later as Claude prompts on unmatched merchants).
3. Markup %? (Default 8.)
4. Tax %? (Default 18. Use 0 for non-tax jurisdictions.)
5. Home currency? (Default INR.)
6. Primary bill-to block: legal name, address lines, city, state, pin, tax ID + tax ID label.
7. Alternate bill-to block? (Optional.)
8. Initial exclusion list? (Optional; the user can add as Claude flags rows.)

Save to YAML. Confirm path. Begin Layer 1.

## What this skill does not do

- It does not categorize for you. You define the buckets and the merchant rules.
- It does not pick the markup. The markup is a business decision; you supply the number.
- It does not file invoices, send emails, or post journal entries. It produces clean CSVs that paste into whatever invoice or accounting software you use.
- It does not learn new exclusion rules without your approval. Every new merchant prompts a confirmation.

The decisions are yours. The typing is the skill's.

## Reference

This skill was extracted from a real monthly-close workflow that replaced ~15 days of manual work with a ~1.5-page document. Story and rationale: see the Medium article in the README at the project root.

Verified for three months side-by-side against the manual workflow before the manual version was retired. No discrepancies in production runs after the switchover.

## Credits

Built by Rehearsal AI. Free to use, modify, and republish. If you ship an improvement, a PR or a note back to the lab is appreciated but not required.
