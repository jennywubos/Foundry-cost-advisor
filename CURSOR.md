# CURSOR.md — Azure AI Foundry PTU Commitment Advisor (`v1/`)

## Project purpose

Single-file static web app (`v1/index.html`) for **Microsoft Azure AI Foundry** customers to compare **Serverless Pay-as-you-go (PAYG)** token usage costs with **Provisioned Throughput Unit (PTU)** options: **hourly**, **monthly reservation ($260/PTU/mo)**, and **annual reservation effective rate ($221/PTU/mo)** over a chosen planning horizon, with a **growth scenario** and **cumulative cost / break-even** visualization (Chart.js).

The app **does not estimate PTU count**; it directs users to the **Azure Capacity Planner / calculator** and accepts **PTU count** as user input.

---

## Key design decisions

| Area | Decision |
|------|----------|
| **Location** | `v1/index.html` only — inline CSS/JS, no build, works via file open in Explorer. |
| **Charts** | Chart.js from `https://cdn.jsdelivr.net/npm/chart.js` (CDN). |
| **Claude** | **Serverless-only** — hide PTU steps; no PTU pricing paths. |
| **TPM helper** | Spread monthly token millions over **full calendar month minutes**: `(mm × 1e6) / (30 × 24 × 60)` (updated from an earlier 8×22 day assumption in the original spec). |
| **Serverless display** | Show **current month** serverless estimate as soon as **both** monthly token fields are valid; do not block on PTU. |
| **PTU comparison** | Full four-way comparison (PAYG vs hourly vs monthly res vs annual res) + break-even plugin **only when PTU count** is valid (normalized to min 15, step 5 for math). |
| **Partial state** | Valid tokens, no PTU: PAYG card shows horizon total; PTU cards `--`; chart shows **PAYG cumulative line only**. |
| **Live updates** | `render()` on `input` / `change` / `keyup` for numeric fields; model `change`/`input`; growth slider `input`; deployment + horizon radios. |
| **Parsing** | Prefer `input.valueAsNumber` for `type="number"` when finite and ≥ 0; else trimmed string with commas stripped. |

---

## Confirmed pricing data and sources (as encoded in the app)

**Do not change these numbers without an explicit product/legal update.**

| Item | Value | Notes |
|------|--------|--------|
| GPT-4.1 | $2.00 / $8.00 per 1M in/out | PTU-supported |
| GPT-4.1 mini | $0.40 / $1.60 | PTU-supported |
| GPT-4.1 nano | $0.10 / $0.40 | PTU-supported |
| DeepSeek-R1 | $0.55 / $2.19 | UI note: indicative — verify in portal |
| Llama-3.3-70B | $0.77 / $0.77 | UI note: indicative — verify in portal |
| Claude Sonnet 4.5 | $3.00 / $15.00 | Serverless-only; **no PTU** |
| Global PTU | **$1.00** / PTU / hour | |
| Data Zone PTU | **$1.10** / PTU / hour | |
| Regional (T1) PTU | **$2.00** / PTU / hour | |
| Monthly reservation | **$260** / PTU / month | Shown same for deployment rows in reference table |
| Annual reservation effective | **$221** / PTU / month equivalent | Footer: effective **Nov 1, 2024** |
| PAYG totals | Compound monthly **base × (1+g)^monthIndex** summed over horizon (`monthIndex` 0-based for first month) | Matches “growth on prior month baseline” formulation |
| PTU hourly total | `ptu × hourlyRate × 24 × 30 × months` | 24×7 billing |
| Reservation totals | Monthly: `ptu × 260 × months`; Annual track: `ptu × 221 × months` (for horizon ≥ 12 months branch) |

**Footer copy (truth in labeling):** “May 2026” publish framing; Calculator requirement; azure.microsoft.com for verification on indicative models.

**Authority:** Conversation-sourced Foundry/public rate snapshot; **`CURSOR.md` is not legal pricing** — always verify live Azure/list pages before procurement.

---

## What NOT to change (unless explicitly tasked)

1. **Hard-coded dollar figures** in the model list and PTU reference table (per original constraints).
2. **No client-side PTU sizing** from tokens — keep portal + user-entered PTU.
3. **Single-file constraint** for `v1/index.html` (no new bundler/React/Vue split) unless the project owner migrates deliberately.
4. **Claude** path: must remain serverless-only with PTU steps hidden.
5. **Calculator URL** opens in **new tab** with `noopener`: `https://oai.azure.com/portal/calculator`
6. **Break-even semantics** tied to cumulative PAYG vs cumulative PTU cost tracks — refactor carefully and re-validate charts.

---

## Related prompt archive

See `prompts/` for authoring and fix transcripts:

- `v1-initial-prompt.md`
- `v1-ptu-commitment-advisor.md`
- `v1-fix-default-values.md`
- `v1-fix-serverless-cost.md`
