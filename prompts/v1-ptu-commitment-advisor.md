# PTU Commitment Advisor — full authoring prompt

This document preserves the comprehensive user specification used to scaffold **`v1/index.html`** as the PTU Commitment Advisor (numbered flow, Claude behavior, Chart.js break-even visualization, footer).

---

```text
## ROLE
You are a senior full-stack developer with expertise in 
building enterprise-grade web applications for Microsoft Azure.
You have deep knowledge of Azure AI Foundry pricing models
including Pay-as-you-go (Serverless), PTU (Provisioned
Throughput Units), and Azure Reservations.

## CONTEXT
This tool will be demonstrated to Microsoft Core AI Senior
management, aiming to help customers (internal and external)
decide between commitment models — which is the exact product
problem the team is solving.

## TASK
Build a single-file web application called
"Azure AI Foundry: PTU Commitment Advisor"
that helps enterprise customers decide:
(1) How many PTUs they need (guided via Azure portal)
(2) Whether to commit to hourly / monthly / annual PTU,
    versus staying on Serverless Pay-as-you-go
(3) When PTU commitment breaks even vs Serverless,
    given expected usage growth over time

## DESIGN PRINCIPLES
- PTU sizing requires Microsoft's official Capacity 
  Calculator. This app does NOT estimate PTU count.
  Instead, it guides users to get the number from 
  Azure portal, then enter it here.
- All cost calculations use only confirmed 
  official Azure pricing — no estimates.
- The app shows a growth scenario analysis:
  as token usage grows over time, when does 
  PTU commitment become cheaper than PAYG?

## IT SHOULD HAVE:

### STEP 1 — Model & Current Token Usage
Header: "Step 1: Enter your current monthly usage"

Model selector dropdown. DO NOT change these prices:

PTU-SUPPORTED models:
- GPT-4.1
  input $2.00/1M | output $8.00/1M
- GPT-4.1 mini
  input $0.40/1M | output $1.60/1M
- GPT-4.1 nano
  input $0.10/1M | output $0.40/1M
- DeepSeek-R1
  input $0.55/1M | output $2.19/1M
  (note: "Azure Foundry price indicative — verify in portal")
- Llama-3.3-70B
  input $0.77/1M | output $0.77/1M
  (note: "Azure Foundry price indicative — verify in portal")

SERVERLESS-ONLY:
- Claude Sonnet 4.5
  input $3.00/1M | output $15.00/1M | NO PTU
  When selected: hide Steps 2-4 entirely.
  Show: "Claude models are billed through Azure 
  Marketplace and do not support PTU.
  Serverless pricing only."

Token usage inputs (oninput, instant update):
- Monthly Input Tokens (millions), default: 100
- Monthly Output Tokens (millions), default: 20

Show immediately below:
"Current monthly Serverless cost: $[calculated]"

### STEP 2 — Get Your PTU Count
Header: "Step 2: Calculate your PTU count in Azure portal"

Show this as a prominent, styled instruction card:

┌─────────────────────────────────────────────┐
│  To get your PTU count:                     │
│                                             │
│  1. Open Azure AI Foundry portal            │
│  2. Go to: Shared resources →               │
│     Model Quota → Azure OpenAI Provisioned  │
│  3. Click "Capacity Planner"                │
│  4. Enter your Input TPM and Output TPM     │
│  5. Click Calculate                         │
│  6. Enter the recommended PTU count below   │
│                                             │
│  [→ Open Azure Capacity Calculator ↗]       │
│  (link: https://oai.azure.com/portal/       │
│   calculator — opens in new tab)            │
└─────────────────────────────────────────────┘

Show helper text below the card:
"Your Input TPM = [calculated from Step 1 inputs] 
 Your Output TPM = [calculated from Step 1 inputs]
 Copy these numbers into the Azure Capacity Calculator."

Calculate and show:
input_TPM = (monthly_input_millions × 1,000,000) 
            / (hours_per_day × 60 × days_active)
            [assume 8 hours/day, 22 days/month as default]
output_TPM = same formula for output tokens

PTU input field:
- PTU Count (from Azure Capacity Calculator)
  default: 100, min: 15, step: 5

Deployment Type radio:
- Global PTU       $1.00/PTU/hour  (default)
- Data Zone PTU    $1.10/PTU/hour
- Regional PTU     $2.00/PTU/hour

Show pricing reference table (read-only):
| Deployment    | Hourly   | Monthly Res. | Annual Res. |
|---------------|----------|--------------|-------------|
| Global        | $1.00/hr | $260/PTU/mo  | $221/PTU/mo |
| Data Zone     | $1.10/hr | $260/PTU/mo  | $221/PTU/mo |
| Regional T1   | $2.00/hr | $260/PTU/mo  | $221/PTU/mo |

Show amber warning:
"⚠️ PTU is billed 24/7 when deployed,
regardless of actual usage hours."

### STEP 3 — Growth Scenario
Header: "Step 3: Set your expected growth"

Two inputs:
- Monthly token growth rate: slider 0% to 30%, 
  step 5%, default 10%
  Label: "Expected monthly usage growth rate"
- Planning horizon: radio buttons
  6 months | 12 months | 24 months (default: 12)

### STEP 4 — Cost Comparison Results
Header: "Step 4: Your cost comparison"

Show 4 cost cards in a row:

Card A — Serverless PAYG (total over planning horizon):
Sum of monthly PAYG costs with growth applied:
month_cost(n) = base_cost × (1 + growth_rate)^n
total = sum of all monthly costs
Label: "Usage-based. Cost grows with traffic."

Card B — PTU Hourly (total over planning horizon):
= ptu_count × hourly_rate × 24 × 30 × months
Label: "Dedicated capacity. Billed 24/7."

Card C — PTU Monthly Reservation:
= ptu_count × 260 × months
Label: "1-month commitment. Flexible."

Card D — PTU Annual Reservation:
= ptu_count × 221 × 12
(only show if planning horizon ≥ 12 months)
Label: "Best rate. 12-month commitment."

Rules:
- Highlight cheapest card: green border + "✓ Best Value"
- Show savings vs PAYG for each PTU option:
  e.g. "Saves $X vs Serverless over 12 months"
- All cards update instantly on any input change

### STEP 5 — Break-Even Chart
Header: "Cumulative cost over time"

Line chart using Chart.js CDN:
https://cdn.jsdelivr.net/npm/chart.js

X-axis: months (1 to planning horizon)
Y-axis: cumulative cost in USD

4 lines:
- Serverless PAYG (curves upward with growth)
- PTU Hourly (straight line)
- PTU Monthly Reservation (straight line, lower)
- PTU Annual Reservation (straight line, lowest — 
  only if planning horizon ≥ 12 months)

Mark the break-even points where PTU lines 
cross the PAYG line with a dot and label:
"Break-even: Month [N]"

Below chart, show text summary:
"With [X]% monthly growth over [Y] months:
· PTU Monthly Reservation breaks even vs 
  Serverless at month [N]
· PTU Annual Reservation saves $[Z] total 
  vs Serverless [OR] is not recommended 
  for [Y]-month horizon"

### STEP 6 — Footer
"Prices based on Azure AI Foundry published rates,
May 2026. PTU reservation pricing effective 
November 1, 2024. PTU count must be obtained from
the Azure AI Foundry Capacity Calculator.
DeepSeek and Llama prices are indicative —
verify current rates at azure.microsoft.com 
before purchasing."

## FORMAT
- Create folder v1 if it does not exist,
  save as v1/index.html
- Single file, all CSS and JS inline
- Only external: Chart.js from CDN
- Dark theme, Microsoft blue accent #0078D4
- Font: Segoe UI, system-ui, sans-serif
- Numbered steps (1-4) make the flow clear
- Max-width 960px, centered
- Professional enterprise feel,
  NOT consumer/startup aesthetic

## CONSTRAINTS
- Do NOT estimate PTU count from token inputs
- Do NOT change any pricing numbers in this prompt
- Do NOT use React, Vue, or any JS framework
- Do NOT create separate CSS or JS files
- All results update instantly via oninput/onchange
- PTU steps completely hidden for Claude
- Azure Capacity Calculator link opens in new tab
- Page works by double-clicking v1/index.html
  in Windows Explorer, no build step needed

## SUCCESS CRITERIA
- All cost calculations are mathematically correct
- Growth curve and break-even analysis are accurate
- Chart shows 4 lines with break-even markers
- Azure Capacity Calculator guidance is prominent
- Claude shows serverless-only correctly
- Looks professional for Microsoft senior management
- Non-technical executive understands the 
  recommendation without explanation
- A customer with $10M/month spend can use this 
  to decide whether to commit to annual PTU
```

**Implementation note:** The original prompt specified TPM using **8 hours/day × 22 days/month**. A later prompt changed TPM to **`(mm × 1e6) / (30 × 24 × 60)`**. The shipped `v1/index.html` reflects the latter.
