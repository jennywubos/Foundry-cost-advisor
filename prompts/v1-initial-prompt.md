# v1 — Initial prompt (Azure AI Foundry cost tool)

Historical note: In the authoring conversation the deliverable was specified as **“Azure AI Foundry: PTU Commitment Advisor”** — a customer-facing advisory UI to compare Pay-as-you-go Serverless versus PTU hourly and reservation economics. Early framing sometimes referred loosely to optimizing Foundry inference cost; the scaffolded artifact is **`v1/index.html`**.

The opening request bundled role/context, numbered steps (model & usage → portal PTU sizing → growth → comparisons → cumulative chart → footer), enforced official pricing only with portal-sourced PTU counts, Claude as serverless-only, Chart.js CDN, dark Microsoft-blue theme, and single-file/no-build constraints.

---

## Verbatim excerpt (opening request summary)

Roles: senior Azure full‑stack engineer with Foundry pricing knowledge.

**Task:** Build a single‑file web app **“Azure AI Foundry: PTU Commitment Advisor”** helping enterprises decide:

1. How many PTUs they need (**no in-app sizing** — guide to Azure Capacity Planner / calculator).
2. Whether to commit to hourly / monthly / annual PTU versus Serverless PAYG.
3. When PTU breaks even vs Serverless given usage growth.

**Design:** PTU sizing from Microsoft’s Capacity Calculator only; official pricing unchanged; growth scenario totals and cumulative Chart.js lines with break-even markers; Claude hides PTU steps; TPM helper from Step 1 for portal entry.

**Detailed specification:** archived in **`prompts/v1-ptu-commitment-advisor.md`** (full STEP 1–6 text, pricing table constraints, FORMAT, SUCCESS CRITERIA).
