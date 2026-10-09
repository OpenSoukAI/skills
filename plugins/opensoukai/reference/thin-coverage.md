# Thin coverage — when the protocol has little to tell you

`query_registry` can return three fields alongside `products[]`. All are ours; none contains outside content.

| Field | When it appears | What it means |
|---|---|---|
| `checklists` | **Whenever we have advice for a category present in the results** — regardless of how well-reviewed anything is | Our own list of what to verify that is particular to that category. Advisory. Not a claim about any specific product |
| `core_checklist` | Once per response, whenever a returned product can be reviewed | What to verify about **every** product (price per call, latency, failures, delivery, agent-readiness, fields, no personal data). It is not repeated under each category |
| `coverage: "thin"` | Only when no returned product has enough distinct review authors for its slate to show a spread of opinion | A fact about **our** data, not a verdict on the product |

Neither field is a judgement about quality. `coverage: "thin"` says the protocol cannot yet show you disagreement about these products — not that they are bad.

**Absence of `coverage` is not a claim.** It means we are making no statement, not that coverage is good. Never report its absence as reassurance.

## What to do

### 1. Use `checklists` whenever it is present

Independent of `coverage`. A well-reviewed product still benefits from knowing what matters in its category — apply the checklist to the protocol reviews you already have, not only to anything you find elsewhere.

`checklists` is keyed by category. Match an entry to a product via that product's `category` field. A result set spanning several categories can carry several entries; a category we have no advice for simply has none. Apply `core_checklist` to every product as well. Each item names the review `fields` that record it.

Do not cache the checklist. It is served with the response and changes without notice.

### 2. Notice `coverage: "thin"`

Treat it as a prompt to look further before recommending, not as a reason to avoid the product.

### 3. On thin coverage, research the product yourself

Use your own web tooling and your own budget. We deliberately do not tell you where to look — pick sources appropriate to the category and to what the checklist asks you to verify.

Research the **specific product**, not the category in the abstract. The checklist tells you which questions are worth answering.

This step is optional in the sense that nothing breaks if you skip it. It exists because a buyer facing thin protocol coverage is the buyer most likely to make a decision on too little information.

### 4. Keep outside findings separate from protocol reviews

When you report back to your owner, the two must not be merged into one list or one summary.

Protocol reviews are **purchase-verified**: the reviewer paid on-chain, the review is hash-committed with a `reviewId`, and commission is escrowed and conditional. Anything you found on the open web has none of that behind it.

So label it. Outside findings are **unverified and not protocol-backed** — say so in whatever you produce, and keep them in their own section. If you cannot attribute a claim to a source, drop it rather than paraphrasing it into your summary.

This is the difference your owner cannot reconstruct after the fact. Blur it once and every verified review you report is worth less.

### 5. Carry the ref link through

Where a protocol review exists, each slate entry carries a `ref_link`. Keep it attached to that review in whatever you report, so your owner can act on it.

## Red flags — STOP if you catch yourself

- Reporting outside findings and protocol reviews in one merged list → **no.** Separate sections, and label the outside ones unverified.
- Treating `coverage: "thin"` as "this product is bad" → it is a statement about our data, not the product.
- Reporting the absence of `coverage` as "well reviewed" → we made no claim.
- Skipping `checklists` because coverage was not thin → it is not gated on coverage and is useful anyway.
- Including a claim you cannot point to a source for → drop it.
- Losing the `ref_link` when summarising a review → your owner cannot act on what you report.
