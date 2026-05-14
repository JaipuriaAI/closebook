# finance-billing

A public Claude skill from **Rehearsal AI**.

A five-layer recipe that turns a monthly credit card or bank statement PDF into reimbursement-ready CSVs — categorized, totaled, marked-up, taxed, bill-to blocks attached, and self-verified before it ever reaches you.

You define the buckets, the merchant rules, the markup %, the tax %, and the bill-to blocks once. The recipe runs the same way every month after.

---

## Quick install

User-scope (available across all your Claude Code projects):

```bash
mkdir -p ~/.claude/skills/finance-billing && \
  curl -o ~/.claude/skills/finance-billing/SKILL.md \
  https://finance-billing-bice.vercel.app/SKILL.md
```

Project-scope (only for the current project):

```bash
mkdir -p .claude/skills/finance-billing && \
  curl -o .claude/skills/finance-billing/SKILL.md \
  https://finance-billing-bice.vercel.app/SKILL.md
```

Then, in any Claude Code session:

> *Process this statement with the finance-billing skill: path-to-statement.pdf*

On first run the skill walks you through a one-time config wizard (buckets, merchant rules, markup %, tax %, bill-to blocks, exclusion list) and saves it to `finance-billing-config.yaml` in the working directory.

---

## What's in this repo

```
.
├── skill/
│   └── SKILL.md           ← the skill itself (the deliverable)
├── landing/
│   ├── index.html         ← Vercel-deployable landing page
│   ├── SKILL.md           ← mirror, served as a static file by Vercel
│   └── vercel.json        ← static config (markdown content-type, CORS, cache)
├── LICENSE                ← MIT
└── README.md              ← this file
```

---

## Deploying the landing page to Vercel

```bash
cd landing
npx vercel --prod
```

Follow the prompts. Vercel detects the static folder and deploys with zero config.

The live deployment is at https://finance-billing-bice.vercel.app.

To attach a custom domain (e.g. `finance-billing.yourdomain.com`):

1. Add the domain in the Vercel project settings.
2. Add the matching DNS record at your domain registrar.
3. Edit `landing/index.html` and `README.md` to swap the Vercel URL for your domain.
4. Commit and push — Vercel auto-redeploys from the GitHub integration.

### Updating the skill later

Edit `skill/SKILL.md`, then:

```bash
cp skill/SKILL.md landing/SKILL.md   # mirror to the landing folder
cd landing && npx vercel --prod
```

---

## Submitting to skills.sh

skills.sh is the community marketplace for Claude skills. Check https://skills.sh for the current submission flow.

Metadata bundle (ready to paste):

| Field | Value |
|-------|-------|
| **Name** | `finance-billing` |
| **Tagline** | Turn a monthly statement PDF into reimbursement-ready CSVs. Self-verified. |
| **Category** | Finance / Productivity / Small Business |
| **Tags** | finance, accounting, monthly-close, expense-management, reimbursement, gst, invoicing, csv, pdf-parsing, india |
| **Install URL** | `https://finance-billing-bice.vercel.app/SKILL.md` |
| **Author** | Rehearsal AI |
| **License** | MIT |
| **Source** | Link to this GitHub repo |

Install command (paste-and-run, suitable for the marketplace listing):

```bash
mkdir -p ~/.claude/skills/finance-billing && \
  curl -o ~/.claude/skills/finance-billing/SKILL.md \
  https://finance-billing-bice.vercel.app/SKILL.md
```

---

## How it works — the five layers

The skill does exactly five things, in order. Skip any one and you have a parlour trick. Build all five — especially the last — and you have a workflow you can run a monthly close on.

1. **Extract** — parse the PDF, take every debit, drop every credit. Apply your exclusion list.
2. **Categorize** — assign each row to one of your buckets via your merchant rules. Unmatched merchants prompt a confirmation and get remembered.
3. **Math** — per bucket: `Reimbursement = sum`, `Markup = Reimbursement × markup_pct`, `Tax = Markup × tax_pct`, `Grand Total = Reimbursement + Markup + Tax`.
4. **Bill-to blocks** — attach the legal name, address, and tax ID you supplied once. Never retyped.
5. **Self-verification** — ten arithmetic + integrity checks run against the skill's own output before anything is shown to you. If any check fails, no CSV ships.

Read the full breakdown in [`skill/SKILL.md`](./skill/SKILL.md).

---

## What this skill is not

- It does not categorize for you. You define the buckets and the merchant rules.
- It does not pick the markup. The markup is a business decision; you supply the number.
- It does not file invoices, send emails, or post journal entries. It produces clean CSVs that paste into whatever invoice or accounting software you use.
- It does not learn new exclusion rules without your approval. Every new merchant prompts a confirmation.

The decisions are yours. The typing is the skill's.

---

## Defaults

- **Markup**: 8% (configurable per business)
- **Tax**: 18% (India GST on services — set to 0 for non-tax jurisdictions, or any other number)
- **Home currency**: INR (configurable)
- **Buckets**: none — defined by you at first run
- **Exclusion list**: empty — grows as you tell the skill about rows it should drop

All of this lives in `finance-billing-config.yaml` in your working directory. Nothing is hard-coded into the skill. Nothing leaves your machine.

---

## License

MIT — free to use, modify, and republish. See [LICENSE](./LICENSE).

If you ship an improvement, a PR is appreciated but not required.

---

## Credits

Built by **Rehearsal AI**. Extracted from a real monthly-close workflow and verified for three months side-by-side against the manual workflow before the manual version was retired. No discrepancies in production runs after the switchover.
