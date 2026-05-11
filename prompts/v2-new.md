# Cursor Prompt: v2/index.html — COMPLETE REWRITE

Completely rewrite `v2/index.html` from scratch as a single-file app (pure HTML + inline CSS + inline JS, no npm, no build tools, no external JS libraries).
Deploy target: `https://jennywubos.github.io/Foundry-cost-advisor/v2/`

---

## WHAT THIS APP IS

A Microsoft field tool: simulates Azure AI Foundry login, loads customer usage data (mocked — APIs commented out), then recommends Priority Processing or PTU based on derived logic. Shows a 12-month cost comparison with real Azure pricing.

---

## VISUAL DESIGN

Microsoft Fluent 2 Dark — exact Azure AI Foundry portal aesthetic.
- Font: `'Segoe UI Variable', 'Segoe UI', system-ui, sans-serif`
- Background: `#1a1a2e`, Surface: `#16213e`, Card: `#0f3460`
- Azure Blue: `#0078d4`, Hover: `#106ebe`, Success: `#107c10`
- Warning: `#ca5010`, Danger: `#a4262c`, Text: `#ffffff`, Secondary: `#c8c6c4`
- Border: `rgba(255,255,255,0.08)`
- Microsoft top bar: `#0078d4` with inline 4-square SVG logo + "Azure AI Foundry" text

---

## PRICING CONSTANTS — FROM AZURE OFFICIAL PRICING PAGE

Source: https://azure.microsoft.com/en-us/pricing/details/cognitive-services/openai-service/
Model: GPT-4.1-2025-04-14 Global

```javascript
const PRICING = {
  // GPT-4.1 Global — Standard tier
  standard: {
    inputPer1M:        2.00,   // $2.00 per 1M input tokens
    cachedInputPer1M:  0.50,   // $0.50 per 1M cached input tokens (75% discount)
    outputPer1M:       8.00,   // $8.00 per 1M output tokens
  },
  // GPT-4.1 Global — Priority Processing tier
  // Source: https://azure.microsoft.com/en-us/pricing/details/cognitive-services/openai-service/
  priority: {
    inputPer1M:        3.50,   // $3.50 per 1M input tokens
    cachedInputPer1M:  0.88,   // $0.88 per 1M cached input tokens
    outputPer1M:      14.00,   // $14.00 per 1M output tokens
  },
  // PTU — capacity-based, not token-based
  ptu: {
    perPTUperMonth:    260,    // $260/PTU/month (monthly reservation)
    annualDiscount:    0.15,   // 15% off for annual prepay
    // Source: https://azure.microsoft.com/en-us/pricing/details/cognitive-services/openai-service/
    // PTU for GPT-4.1: input TPM per PTU = 2,500 (verified from Foundry calculator)
    // Output token ratio: 1 output token = 4 input tokens (GPT-4.1, per Microsoft docs)
    // Source: https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/provisioned-throughput-onboarding
    tpmPerPTU_input:   2500,
    outputRatio:       4,      // 1 output token = 4 input tokens toward PTU utilization
  }
};
```

---

## COST CALCULATION FUNCTIONS — ALL COSTS GENERATED, NEVER HARDCODED

```javascript
// Calculate effective monthly cost given token volumes and cache hit rate
function calcMonthlyCost(monthlyInputM, monthlyOutputM, cacheRate, tier) {
  const p = PRICING[tier];
  const cachedInputM  = monthlyInputM * cacheRate;
  const uncachedInputM = monthlyInputM * (1 - cacheRate);
  if (tier === 'ptu') {
    // PTU is capacity-based — cost is fixed regardless of token volume
    return PRICING.ptu.perPTUperMonth * calcRecommendedPTUs(monthlyInputM, monthlyOutputM);
  }
  return (uncachedInputM * p.inputPer1M)
       + (cachedInputM   * p.cachedInputPer1M)
       + (monthlyOutputM * p.outputPer1M);
}

// Calculate recommended PTUs from Foundry calculator formula
// Source: https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/provisioned-throughput-onboarding
function calcRecommendedPTUs(monthlyInputM, monthlyOutputM) {
  // Convert monthly tokens → peak TPM (assume 720 hours/month for 24/7, use peak hour)
  // Use the PTU calculator logic: peak TPM / tpmPerPTU, with 17% headroom buffer
  const peakInputTPM  = (monthlyInputM  * 1e6) / (720 * 60);  // average TPM from monthly
  const peakOutputTPM = (monthlyOutputM * 1e6) / (720 * 60);
  const effectiveInputTPM = peakInputTPM + (peakOutputTPM * PRICING.ptu.outputRatio);
  const basePTUs = effectiveInputTPM / PRICING.ptu.tpmPerPTU_input;
  return Math.ceil(basePTUs * 1.17); // 17% headroom buffer
}

// 12-month projection with 10% MoM growth — FULLY GENERATED
function calc12MonthProjection(baseInputM, baseOutputM, cacheRate) {
  return Array.from({length: 12}, (_, i) => {
    const g = Math.pow(1.10, i); // 10% compound growth
    const inputM  = baseInputM  * g;
    const outputM = baseOutputM * g;
    const ptuPTUs = calcRecommendedPTUs(baseInputM, baseOutputM); // PTU sized to base, not re-sized
    return {
      month: i + 1,
      standard:    calcMonthlyCost(inputM, outputM, cacheRate, 'standard'),
      priority:    calcMonthlyCost(inputM, outputM, cacheRate, 'priority'),
      ptuMonthly:  ptuPTUs * PRICING.ptu.perPTUperMonth,
      ptuAnnual:   ptuPTUs * PRICING.ptu.perPTUperMonth * (1 - PRICING.ptu.annualDiscount),
    };
  });
}
```

---

## OFFER CONTEXT — SOURCED FROM MICROSOFT DOCS, NOT HARDCODED STRINGS

```javascript
// These strings are sourced verbatim/paraphrased from official Microsoft documentation.
// In production, fetch from Microsoft docs API. In demo: inline with source citations.
const OFFER_CONTEXT = {
  priorityProcessing: {
    designTarget: "Pay-as-you-go simplicity with no long-term commitments. Business-hour or bursty traffic that benefits from scalable, cost-efficient performance.",
    quotaBehavior: "Priority processing uses the same quota as standard processing — no separate capacity pool. Requests are routed with higher priority than Standard in the shared queue.",
    latencyBenefit: "Consistent, low latency for responsive user experiences (p50 latency SLA per 5-minute window).",
    limitation: "Long context requests exceeding 128k prompt tokens are downgraded to standard processing at standard rates.",
    sources: [
      { label: "Enable priority processing — Microsoft Learn", url: "https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/priority-processing" },
      { label: "What's new in Microsoft Foundry Oct–Nov 2025", url: "https://devblogs.microsoft.com/foundry/whats-new-in-microsoft-foundry-oct-nov-2025/" },
    ]
  },
  ptu: {
    designTarget: "Provisioned deployments are recommended for workloads with consistent usage that require predictable latency and throughput.",
    sla: "PTU provides a dedicated GPU lane: latency and throughput are guaranteed within purchased capacity. Traffic exceeding purchased PTUs overflows to standard (pay-as-you-go) at standard rates.",
    cacheBenefit: "For Provisioned deployment types, cached tokens receive up to 100% discount on input tokens — enabling up to 2× throughput at the same PTU cost when 50% cache match rate is achieved.",
    billingModel: "Billing is based on PTU-hours, not tokens — cost is fixed regardless of token volume. Monthly, annual (15% discount), and hourly options available.",
    sources: [
      { label: "Provisioned throughput onboarding — Microsoft Learn", url: "https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/provisioned-throughput-onboarding" },
      { label: "Azure OpenAI pricing", url: "https://azure.microsoft.com/en-us/pricing/details/cognitive-services/openai-service/" },
    ]
  },
  cacheNote: {
    globalStandardWarning: "On Global Standard deployments, prompt caching hit rates are typically low (1–5%) due to dynamic request routing across global infrastructure. Cache hits are not guaranteed — requests are routed based on a hash of the prompt prefix and may land on different servers with empty caches.",
    // Source: https://learn.microsoft.com/en-us/answers/questions/2154181/azure-openai-api-caching-issue-with-model-gpt-4o-m
    // Real user report: "Cache hit is not a promise. Cache hit can be lower on global Standard tier since dynamic routing is involved."
    provisioned: "PTU deployments retain cache on dedicated infrastructure — up to 100% cache discount and significantly higher hit rates achievable with prefix-stable prompts.",
    sources: [
      { label: "Prompt caching — Microsoft Learn", url: "https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/prompt-caching" },
    ]
  }
};
```

---

## RECOMMENDATION ENGINE — FULLY GENERATED FROM DATA + OFFER_CONTEXT

```javascript
function generateRecommendation(user, data) {
  // Derive characteristics from actual metric values
  const throttleHigh   = data.throttledCallsPct > 0.05;   // >5%
  const throttleMod    = data.throttledCallsPct > 0.01;   // >1%
  const quotaHigh      = (data.quotaAllocatedTPM / data.quotaLimitTPM) > 0.50;
  const cacheVeryLow   = data.cacheMatchRate < 0.05;
  const cacheLow       = data.cacheMatchRate < 0.20;
  const workloadSpiky  = data.workloadPattern === 'spiky';
  const workload247    = data.workloadPattern === '247';

  // Score each offer
  let priorityScore = 0, ptuScore = 0;

  // Priority Processing fit signals
  // Source: OFFER_CONTEXT.priorityProcessing.designTarget
  if (workloadSpiky)   priorityScore += 3;  // "Business-hour or bursty traffic"
  if (throttleHigh)    priorityScore += 3;  // throttle hurting users, needs routing priority
  if (!quotaHigh)      priorityScore += 1;  // not approaching quota limit
  if (cacheLow)        priorityScore += 1;  // PTU cache economics not favorable

  // PTU fit signals
  // Source: OFFER_CONTEXT.ptu.designTarget
  if (workload247)     ptuScore += 3;  // "consistent usage"
  if (quotaHigh)       ptuScore += 3;  // approaching Standard TPM limit
  if (throttleMod)     ptuScore += 1;  // growing throttle signals capacity need
  if (!workloadSpiky)  ptuScore += 1;  // not wasting overnight capacity

  const recommendation = ptuScore > priorityScore ? 'ptu' : 'priority';

  // Generate headline from actual data values
  let headline = '';
  const bullets = [];

  if (recommendation === 'priority') {
    headline = `Your usage pattern — ${(data.throttledCallsPct * 100).toFixed(1)}% throttle rate during business hours with variable overnight volume — matches the design target for Priority Processing: "${OFFER_CONTEXT.priorityProcessing.designTarget}"`;

    if (throttleHigh)
      bullets.push(`${(data.throttledCallsPct * 100).toFixed(1)}% throttled calls (last 30 days) → Standard is blocking users at peak. Priority Processing routes your requests ahead of Standard in the shared queue — no capacity reservation required. (${OFFER_CONTEXT.priorityProcessing.quotaBehavior})`);

    if (workloadSpiky)
      bullets.push(`Spiky workload (business hours only, ~${data.workingHoursPerMonth} hrs/month) → PTU dedicated capacity would sit unused overnight. Priority Processing is PAYG — you only pay for tokens consumed.`);

    if (cacheVeryLow)
      bullets.push(`${(data.cacheMatchRate * 100).toFixed(0)}% cache match rate → consistent with Global Standard behavior: "${OFFER_CONTEXT.cacheNote.globalStandardWarning.split('.')[0]}." Each legal contract is unique, further reducing reuse. PTU economics strengthen significantly above 40% cache rate — not achievable here without workload restructuring.`);

    bullets.push(`Same PAYG billing model as Standard, no commitment — but with p50 latency SLA per 5-minute window. ${OFFER_CONTEXT.priorityProcessing.latencyBenefit}`);
    bullets.push(`No need to forecast monthly token volume for PTU sizing. Priority Processing scales with actual usage — critical for unpredictable caseload.`);

  } else {
    headline = `Your consistent 24/7 workload at ${((data.quotaAllocatedTPM / data.quotaLimitTPM) * 100).toFixed(0)}% quota utilization matches the PTU design target: "${OFFER_CONTEXT.ptu.designTarget}"`;

    if (quotaHigh)
      bullets.push(`${((data.quotaAllocatedTPM / data.quotaLimitTPM) * 100).toFixed(0)}% quota utilization → approaching the Standard TPM rate limit ceiling. PTU provides dedicated capacity outside the shared quota pool, eliminating rate-limit risk as usage grows.`);

    if (workload247)
      bullets.push(`24/7 consistent load → PTU dedicated GPU capacity is fully utilized around the clock. Unlike Standard, reserved throughput is never idle. ${OFFER_CONTEXT.ptu.billingModel}`);

    if (cacheVeryLow) {
      const projectedRPM = Math.round(data.peakRPM * 2);
      bullets.push(`Current cache match rate: ${(data.cacheMatchRate * 100).toFixed(0)}% → opportunity: "${OFFER_CONTEXT.ptu.cacheBenefit}" A support chatbot with stable FAQ system prompts is a strong candidate for 40–50% cache rates on PTU, potentially doubling throughput to ~${projectedRPM} RPM at zero additional cost. (Source: Chris Hoder, Azure AI Foundry Ignite 2025 demo)`);
    }

    const ptus = calcRecommendedPTUs(data.monthlyInputTokensM, data.monthlyOutputTokensM);
    bullets.push(`Foundry PTU calculator recommends ${ptus} PTUs for ${(data.peakInputTPM / 1e6).toFixed(2)}M peak input TPM → $${(ptus * PRICING.ptu.perPTUperMonth).toLocaleString()}/month fixed cost. Not exposed to token price growth as volume scales.`);

    bullets.push(`${OFFER_CONTEXT.ptu.sla}`);
  }

  return { recommendation, headline, bullets, sources: OFFER_CONTEXT[recommendation === 'ptu' ? 'ptu' : 'priorityProcessing'].sources };
}
```

---

## TWO DEMO CUSTOMERS

### Customer A — Sarah Chen → Priority Processing
```javascript
const DEMO_USERS = {};

DEMO_USERS["sarah.chen@contoso.com"] = {
  password: "Demo2024!",
  name: "Sarah Chen",
  initials: "SC",
  company: "Contoso Legal",
  subscriptionId: "a1b2c3d4-xxxx-xxxx-xxxx-contoso0001",
  subscriptionName: "Contoso-Legal-Prod",
  resourceGroup: "rg-contoso-openai",
  accountName: "openai-contoso-eastus",
  deployedModel: "gpt-4.1-2025-04-14",
  region: "East US",
};

const MOCK_DATA_SARAH = {
  workloadPattern: "spiky",            // business hours Mon–Fri
  workingHoursPerMonth: 220,           // 22 days × 10 hours
  peakRPM: 220,
  peakInputTPM: 3344000,               // 220 RPM × 15,200 avg tokens
  peakOutputTPM: 22000,                // 220 RPM × 100 avg tokens
  outputTPM_30dayAvg: 19800,           // slightly lower 30-day average
  avgPromptTokens: 15200,
  avgCompletionTokens: 100,

  // Cache match rate: Global Standard + diverse legal docs = very low
  // Source: Microsoft Q&A — "Cache hit can be lower on global Standard tier since dynamic routing is involved"
  // https://learn.microsoft.com/en-us/answers/questions/2154181/azure-openai-api-caching-issue
  // Real reported range on Global Standard: 0.1%–4%; unique legal contracts push toward lower end
  cacheMatchRate: 0.03,                // 3% — Global Standard + unique documents

  throttledCallsPct: 0.142,            // 14.2% — 30-day avg, peak business hours
  avgLatencyMs: 1400,
  quotaAllocatedTPM: 1000000,
  quotaLimitTPM: 6000000,

  // Monthly token volumes — business hours only (220 hrs/month)
  // 220 RPM × 15,200 tokens × 60 min × 220 hrs / 1,000,000
  monthlyInputTokensM: 44141,
  // 220 RPM × 100 tokens × 60 min × 220 hrs / 1,000,000
  monthlyOutputTokensM: 290,

  // Available capacity — East US, gpt-4.1
  capacity: [
    { model: "gpt-4.1-2025-04-14", type: "Global Provisioned-Managed", availablePTUs: 340, totalPTUs: 500 },
    { model: "gpt-4.1-2025-04-14", type: "Global Standard",            availableTPM: 4200000, totalTPM: 6000000 },
  ],
};
```

### Customer B — James Okonkwo → PTU
```javascript
DEMO_USERS["james.okonkwo@fabricam.com"] = {
  password: "Demo2024!",
  name: "James Okonkwo",
  initials: "JO",
  company: "Fabricam Financial",
  subscriptionId: "b2c3d4e5-xxxx-xxxx-xxxx-fabricam001",
  subscriptionName: "Fabricam-Fin-Prod",
  resourceGroup: "rg-fabricam-openai",
  accountName: "openai-fabricam-eastus",
  deployedModel: "gpt-4.1-2025-04-14",
  region: "East US",
};

const MOCK_DATA_JAMES = {
  workloadPattern: "247",              // 24/7 consistent
  workingHoursPerMonth: 720,           // 24 × 30
  peakRPM: 220,
  peakInputTPM: 3344000,
  peakOutputTPM: 22000,
  outputTPM_30dayAvg: 21500,
  avgPromptTokens: 15200,
  avgCompletionTokens: 100,

  // Cache match rate: Global Standard routing = low; support chatbot has some FAQ repetition
  // But Global Standard dynamic routing suppresses cache. 0% is the current mocked baseline.
  // Source: same Microsoft Q&A reference above
  cacheMatchRate: 0.00,                // 0% — Global Standard, not yet optimized

  throttledCallsPct: 0.021,            // 2.1% — 30-day avg
  avgLatencyMs: 1400,
  quotaAllocatedTPM: 3500000,
  quotaLimitTPM: 6000000,              // 58% utilized — approaching limit

  // Monthly token volumes — 24/7
  // 220 RPM × 15,200 tokens × 60 min × 720 hrs / 1,000,000
  monthlyInputTokensM: 144461,
  // 220 RPM × 100 tokens × 60 min × 720 hrs / 1,000,000
  monthlyOutputTokensM: 950,

  // PTU recommendation (from calcRecommendedPTUs — NOT hardcoded)
  // calcRecommendedPTUs(144461, 950) → ~95 PTUs (shown live in UI)

  // Post-cache opportunity (PTU enables up to 100% cache discount)
  projectedCacheRateOnPTU: 0.50,       // achievable for FAQ-heavy support chatbot on PTU
  projectedRPMWithCache: 437,          // 2× throughput (Chris Hoder, Ignite 2025 demo)
  projectedLatencyMs: 1100,

  capacity: [
    { model: "gpt-4.1-2025-04-14", type: "Global Provisioned-Managed", availablePTUs: 340, totalPTUs: 500 },
    { model: "gpt-4.1-2025-04-14", type: "Global Standard",            availableTPM: 2500000, totalTPM: 6000000 },
  ],
};

// Attach mock data to each user by email
const MOCK_DATA_BY_USER = {
  "sarah.chen@contoso.com":    MOCK_DATA_SARAH,
  "james.okonkwo@fabricam.com": MOCK_DATA_JAMES,
};
```

---

## HISTORY GENERATORS (used for sparklines — GENERATED not hardcoded)

```javascript
function generateSpikeyHistory(d) {
  // Spiky: two busy windows per hour, quiet overnight
  return Array.from({length: 60}, (_, i) => {
    const isBusy = (i >= 5 && i <= 22) || (i >= 35 && i <= 52);
    const isThrottle = i === 14 || i === 48;
    const rpm = isBusy ? d.peakRPM * (0.85 + Math.random()*0.2)
                       : d.peakRPM * (0.05 + Math.random()*0.08);
    return {
      min: i,
      rpm: Math.round(rpm),
      inputTPM: Math.round(rpm * d.avgPromptTokens),
      outputTPM: Math.round(rpm * d.avgCompletionTokens),
      throttlePct: isThrottle ? 0.32 : (isBusy ? 0.08 + Math.random()*0.06 : 0.001),
      cacheRate: d.cacheMatchRate + (Math.random()-0.5)*0.01,
    };
  });
}

function generateSteadyHistory(d) {
  return Array.from({length: 60}, (_, i) => {
    const rpm = d.peakRPM * (0.92 + Math.random()*0.1);
    return {
      min: i,
      rpm: Math.round(rpm),
      inputTPM: Math.round(rpm * d.avgPromptTokens),
      outputTPM: Math.round(rpm * d.avgCompletionTokens),
      throttlePct: d.throttledCallsPct * (0.8 + Math.random()*0.4),
      cacheRate: d.cacheMatchRate,
    };
  });
}
```

---

## UI SECTIONS

### SECTION 0: TOP BAR (always visible after login)
```
[■■ Azure AI Foundry]    [🔴 Demo Mode — Mock Data ⓘ]    [● James Okonkwo ▾  Sign out]
                                                           Fabricam-Fin-Prod (b2c3d4e5-xxxx)
```
Tooltip on 🔴: "Real data available via Azure Monitor API — see commented code in source"

---

### SECTION 1: LOGIN PAGE

Single centered card, Microsoft-style white-on-dark:
```
[Microsoft 4-square SVG]
Azure AI Foundry

Email address     [_________________________________]
Password          [_________________________________]
                  Wrong credentials → red inline error: exact Microsoft error text:
                  "That Microsoft account doesn't exist. Try another."

                  [          Sign in          ]

Demo credentials:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📧 sarah.chen@contoso.com  /  Demo2024!
   → Contoso Legal — Priority Processing demo
📧 james.okonkwo@fabricam.com  /  Demo2024!
   → Fabricam Financial — PTU demo
```

On "Sign in": validate → 1.2s spinner with "Signing you in..." → fade to dashboard.

---

### SECTION 2: USAGE SNAPSHOT — "Your Current Usage — Global Standard | Last 30 days"

6 KPI cards with count-up animation:

| Card | Value | Sub-label |
|---|---|---|
| PEAK RPM | 220 | peak, last hour |
| INPUT TPM (peak) | 3.34M | peak, last hour |
| OUTPUT TPM | 22K | 30-day avg |
| CACHE MATCH RATE | 3% (Sarah) / 0% (James) | 30-day avg |
| THROTTLED CALLS | 14.2% / 2.1% | 30-day avg |
| QUOTA USED | 17% / 58% | 30-day avg |

Color rules:
- CACHE MATCH RATE: red < 10%, yellow 10–30%, green > 30%
- THROTTLED CALLS: green < 2%, yellow 2–10%, red > 10%
- QUOTA USED: green < 50%, yellow 50–80%, red > 80%
- AVG LATENCY not in KPIs (shown in recommendation section)

**Cache Match Rate tooltip** (ⓘ icon):
"On Global Standard deployments, prompt cache hit rates are typically 1–5% due to dynamic routing across global infrastructure. Requests may land on different servers with empty caches. Source: Azure OpenAI documentation"

**Available Capacity** — show ONLY the rows matching customer's deployed model + region:
```
[customer.deployedModel] | Global Provisioned-Managed | ████████░░░  340 / 500 PTUs available
[customer.deployedModel] | Global Standard             | ███████░░░░  X / 6M TPM available
```
Note: "Showing capacity for your model (gpt-4.1-2025-04-14) in East US"

**SVG Sparkline** — 60-minute history, drawn in pure JS:
- Lines: INPUT TPM (blue #0078d4), OUTPUT TPM (teal #00b294), THROTTLE % × scale (red #a4262c)
- Sarah: spiky pattern with two busy windows + throttle spikes at minutes 14 and 48
- James: flat steady line
- X-axis label: "60 minutes ago" ← → "now"
- Red vertical tick marks where throttle > 20%

---

### SECTION 3: COMMITMENT ADVISOR

**Recommendation banner** — generated by `generateRecommendation()`:
```
[⚡ or 🏦] RECOMMENDED: [Priority Processing / Provisioned PTU]
[generated headline — reads actual throttle %, quota %, workload pattern]
```

**Why [offer] fits [company name]** — generated bullets from `generateRecommendation().bullets`
- Every bullet references actual customer metric values
- Every bullet that references offer behavior includes the source

**Source links** at bottom — from `generateRecommendation().sources`
Rendered as: `Sources: [label] (url) · [label] (url)`

---

### SECTION 4: THREE-COLUMN COMPARISON + 12-MONTH PROJECTION

**Three cards** — costs generated by `calcMonthlyCost()`:

```
STANDARD (PAYG)          PRIORITY PROCESSING        PROVISIONED PTU
Current plan             [RECOMMENDED* / not]        [RECOMMENDED* / not]

$2.00/1M input           $3.50/1M input              $260/PTU/month
$0.50/1M cached input    $0.88/1M cached input        [N] PTUs = $X/mo
$8.00/1M output          $14.00/1M output            [computed by calcRecommendedPTUs()]

❌ No SLA guarantee      ✅ p50 latency SLA          ✅ Dedicated GPU
❌ Throttle risk         ✅ Priority queue            ✅ No throttle
❌ Dynamic routing       ✅ No commitment             ✅ Predictable latency
✅ No commitment         ⚠️ Shared quota pool         ✅ 100% cache discount
✅ Lowest base price     ✅ PAYG flexibility           ⚠️ Commitment required

Monthly est: $[calc]     Monthly est: $[calc]         Monthly est: $[calc]
```

Recommended card: glowing `#0078d4` border + "RECOMMENDED" badge at top.

For James only — PTU card footnote:
"*With 50% cache match rate on PTU (100% cache discount): throughput → [2× RPM] RPM, latency → [projectedLatencyMs]ms. Source: Chris Hoder, Azure AI Foundry Ignite 2025"

**PTU Sizing Calculator** (collapsible, expanded by default for James):
```
How we calculate [N] PTUs:
Peak input TPM:     [peakInputTPM]
Peak output TPM:    [peakOutputTPM] × 4 (output ratio for GPT-4.1)
Effective input TPM:[computed]
TPM per PTU (GPT-4.1 Global Prov-Managed): 2,500
Base PTUs needed:   [base]
+ 17% headroom:     [final] PTUs ✓
Source: Azure AI Foundry PTU calculator
https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/provisioned-throughput-onboarding
```

**12-Month Cost Projection Table** — generated by `calc12MonthProjection()`:

| | Month 1 | Month 6 | Month 12 | 12-Month Total |
|---|---|---|---|---|
| Standard (PAYG) | $[calc] | $[calc] | $[calc] | $[calc] |
| Priority Processing | $[calc] (+X%) | $[calc] | $[calc] | $[calc] |
| PTU (monthly) | $[calc] | $[calc] | $[calc] | $[calc] |
| PTU (annual prepay) | $[calc] (-15%) | $[calc] | $[calc] | $[calc] |

Note: "Token-based rows assume 10% month-over-month growth. PTU is flat capacity pricing — no exposure to token volume growth."

**SVG bar chart** — 4 bars for 12-month total, drawn in pure JS. Color: Standard=grey, Priority=blue, PTU monthly=teal, PTU annual=green.

---

### SECTION 5: FOOTER

```
Data sources: Azure AI Capacity API · Azure Monitor Metrics API · Azure AI Foundry Quota API
I'm intentionally keeping data mocked — the goal is to validate the workflow, not the backend.
Priority Processing: learn.microsoft.com/en-us/azure/foundry/openai/concepts/priority-processing
PTU pricing: azure.microsoft.com/en-us/pricing/details/cognitive-services/openai-service/
v2.0 | github.com/jennywubos/Foundry-cost-advisor | ← v1 (manual input)
```

---

## COMMENTED-OUT REAL API CODE — 3 BLOCKS

Each block placed immediately before the mock data it replaces.

### Block 1: Capacity API
```javascript
/* === REAL API: Azure AI Capacity API ===
 * GET https://management.azure.com/subscriptions/{subscriptionId}
 *     /providers/Microsoft.CognitiveServices/locations/{location}
 *     /models?api-version=2024-10-01
 * Auth: Bearer token — Azure AD / Microsoft Entra ID
 *   In production: MSAL.js PublicClientApplication.loginPopup()
 *   scope: "https://management.azure.com/.default"
 *   (This is exactly how Azure AI Foundry portal authenticates users)
 *
 * async function fetchCapacity(token, subscriptionId, location) {
 *   const res = await fetch(
 *     `https://management.azure.com/subscriptions/${subscriptionId}`
 *     + `/providers/Microsoft.CognitiveServices/locations/${location}`
 *     + `/models?api-version=2024-10-01`,
 *     { headers: { Authorization: `Bearer ${token}` } }
 *   );
 *   const data = await res.json();
 *   // Filter to customer's deployed model only:
 *   return data.value.filter(m => m.name === currentUser.deployedModel);
 * }
 * === END REAL API === */
```

### Block 2: Azure Monitor API
```javascript
/* === REAL API: Azure Monitor Metrics API ===
 * GET https://management.azure.com/subscriptions/{subscriptionId}
 *     /resourceGroups/{resourceGroup}
 *     /providers/Microsoft.CognitiveServices/accounts/{accountName}
 *     /providers/microsoft.insights/metrics
 *     ?api-version=2023-10-01
 *     &metricnames=ProcessedPromptTokens,GeneratedCompletionTokens,
 *                  CacheMatchRate,BlockedCalls,SuccessfulCalls
 *     &timespan=P30D&interval=PT1H&aggregation=Total,Average
 *
 * IMPORTANT: Requires BOTH subscriptionId AND resourceGroup + accountName
 *   (full ARM resource path). subscriptionId alone is NOT sufficient.
 *
 * Metric names for Foundry/Azure OpenAI:
 *   ProcessedPromptTokens      → input tokens (split by deployment via dimensions)
 *   GeneratedCompletionTokens  → output tokens
 *   CacheMatchRate             → prompt cache hit rate (0.0–1.0) — 30-day avg
 *   BlockedCalls               → 429 throttled requests
 *   SuccessfulCalls            → 200 OK
 *
 * async function fetchMonitor(token, subscriptionId, rg, account, timespan='P30D') {
 *   const base = `https://management.azure.com/subscriptions/${subscriptionId}`
 *     + `/resourceGroups/${rg}`
 *     + `/providers/Microsoft.CognitiveServices/accounts/${account}`;
 *   const res = await fetch(
 *     base + `/providers/microsoft.insights/metrics`
 *       + `?api-version=2023-10-01`
 *       + `&metricnames=ProcessedPromptTokens,GeneratedCompletionTokens,CacheMatchRate,BlockedCalls,SuccessfulCalls`
 *       + `&timespan=${timespan}&interval=PT1H&aggregation=Total,Average`,
 *     { headers: { Authorization: `Bearer ${token}` } }
 *   );
 *   return await res.json();
 * }
 * === END REAL API === */
```

### Block 3: Quota API
```javascript
/* === REAL API: Azure AI Foundry Quota API ===
 * Subscription-level quota:
 * GET https://management.azure.com/subscriptions/{subscriptionId}
 *     /providers/Microsoft.CognitiveServices/locations/{location}
 *     /usages?api-version=2024-10-01
 *
 * Deployment-level rate limits:
 * GET https://management.azure.com/subscriptions/{subscriptionId}
 *     /resourceGroups/{rg}
 *     /providers/Microsoft.CognitiveServices/accounts/{account}
 *     /deployments?api-version=2024-10-01
 *   → properties.rateLimits[].count = TPM rate limit per deployment
 *   → properties.sku.capacity = PTUs allocated (for PTU deployments)
 *
 * async function fetchQuota(token, subscriptionId, location) {
 *   const res = await fetch(
 *     `https://management.azure.com/subscriptions/${subscriptionId}`
 *     + `/providers/Microsoft.CognitiveServices/locations/${location}`
 *     + `/usages?api-version=2024-10-01`,
 *     { headers: { Authorization: `Bearer ${token}` } }
 *   );
 *   return (await res.json()).value;
 * }
 * === END REAL API === */
```

---

## TECHNICAL REQUIREMENTS

1. **Single file** `v2/index.html` — inline `<style>` + inline `<script>`. Zero external JS (Segoe UI is a system font, no CDN needed).
2. **No frameworks, no npm.** Pure vanilla JS + HTML + CSS.
3. **Zero hardcoded cost numbers** — every dollar figure computed by `calcMonthlyCost()` or `calc12MonthProjection()`.
4. **Zero hardcoded recommendation text** — every sentence generated by `generateRecommendation()` reading actual data values.
5. **Pricing constants clearly attributed** — every price in `PRICING` has a source URL comment.
6. **OFFER_CONTEXT strings attributed** — every string has a source URL comment pointing to Microsoft docs.
7. **SVG charts** drawn in vanilla JS — no Chart.js, no D3.
8. **Count-up animation** on KPI cards using `requestAnimationFrame`.
9. **Login**: both fields on same card, single "Sign in" button, inline error on wrong credentials.
10. **Responsive** 1280px+ desktop primary.
11. **GitHub Pages ready** — relative paths only.
12. **Write the complete file** — do NOT truncate, do NOT use `// ... rest of code`. Every function fully implemented.

---

## OUTPUT

Completely overwrite `v2/index.html`. No other files created or modified.
