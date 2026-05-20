# SME Credit Triage Engine
### Continuous Underwriting & Dynamic Terms — Interactive Simulation Dashboard

> **Jurisdiction:** Ontario, Canada · FSRA-regulated non-bank lenders & credit unions  
> **Target segment:** SMEs with $150K–$5M annual revenue  
> **Architecture version:** v2.0 — stateful, rule-based, explainable

[![Live Demo](https://img.shields.io/badge/Live%20Demo-GitHub%20Pages-00d4ff?style=flat-square&logo=github)](https://seungaj.github.io/sme-credit-triage-engine/)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)
[![Architecture](https://img.shields.io/badge/Architecture-Rule--Based%20%2B%20EWMA-blue?style=flat-square)]()
[![Jurisdiction](https://img.shields.io/badge/Jurisdiction-Ontario%2C%20Canada-red?style=flat-square)]()

---

## Overview

This repository contains a **fully self-contained, single-file interactive simulation** of a Continuous Underwriting and Dynamic Terms (CU/DT) credit triage engine purpose-built for Ontario SME lenders. It runs entirely in the browser — no backend, no build step, no dependencies to install.

The engine replaces legacy document-heavy underwriting workflows (typically 2–8 weeks) with a continuous, monthly data-driven credit decisioning loop that remains **explainable to underwriters** and **defensible from a regulatory standpoint**.

This is a **proposed architecture simulation**, not a production system. It is intended as a design thinking and demonstration tool.

---

## Live Demo

Open `index.html` directly in any modern browser, or deploy via GitHub Pages (see [Deployment](#deployment)).

Three pre-loaded Ontario SME borrowers demonstrate the full range of credit states:

| Borrower | Sector | Score | Tier | State |
|---|---|---|---|---|
| Maple Brew Co. | Craft Brewery & Taproom | 82 | Green A | Auto-Approved |
| Nova Freight Ltd. | Logistics & Transport | 47 | Yellow B | Underwriter Review |
| Strata Build Inc. | General Contracting | 24 | Red | Draw Freeze |

---

## Architecture

### The Four Problems This Solves

| Legacy Problem | Engine Solution |
|---|---|
| Approvals take 2–8 weeks | Monthly auto-limit on live bank feed data |
| Rigid covenants called abruptly | Graduated monthly price signal with hysteresis protection |
| Forced into expensive private lenders (10–18%+ APR) | Real-time risk pricing fills the gap between bank rates (~6%) and private lenders |
| No reward for financial discipline | Rate decreases fire in the same cycle as score improvement (symmetry rule) |

### 10-Layer Processing Pipeline

```
L1  INGEST      Accounting CSV (QuickBooks/Xero) + bank feed (Plaid/Flinks) + BoC rate
L2  NORMALISE   COA standardisation, LTM + trailing-3m metrics
L3  FORECAST    6–8 week forward liquidity model
L4  STRESS      Base (0%/0%) · Adverse (−15%/+5%) · Severe (−30%/+10%)
L5  SCORE       Composite risk score with EWMA smoothing (α = 0.5)
L6  COVERAGE    PPSA GSA ratio monitoring — internal only, 5-level alert
L7  TERMS       Rate + limit proposal (collateral fixed at origination)
L8  GUARDRAILS  Hard caps, floors, dead-band, cumulative 3m cap, PPSA boundary
L9  ROUTE       AUTO_EXECUTE · UNDERWRITER_REVIEW · SENIOR_ESCALATION · DRAW_FREEZE
L10 AUDIT       41-field immutable log including disclosure_text_hash
```

### Scoring Formula

```
Score(t) = EWMA(raw, prevScore, α=0.5)

raw = 0.25 × RunwayScore
    + 0.25 × DSCRScore
    + 0.20 × ARHealthScore
    + 0.15 × PaymentScore
    + 0.15 × StressShortfallScore

RunwayScore       = min(100, (WeeksRunway / 12) × 100)
DSCRScore         = min(100, (DSCR / 2.0) × 100)
ARHealthScore     = 100 − (AR_90plus × 2) − (AR_60 × 1)  [floor 0]
PaymentScore      = 100 − (DaysLate × 3) − (NSF × 10)    [floor 0]
StressShortfallScore = 100 − (SevereShortfall_weeks × 10) [floor 0]
```

**Sub-segment calibration ($150K–$499K band):** DSCR receives a +0.15 owner compensation normalisation adjustment before scoring.

### Risk Tiers

| Tier | Score Range | PD Proxy | Tier Premium |
|---|---|---|---|
| Green A | 80–100 | < 1% | 100 bps |
| Green B | 65–79 | 1–3% | 175 bps |
| Yellow A | 50–64 | 3–8% | 325 bps |
| Yellow B | 35–49 | 8–18% | 500 bps |
| Red | 0–34 | 18%+ | 800 bps |

### Interest Rate Formula

```
Rate(t) = 550 bps (BoC + spread)
        + TierPremium(activeTier)
        + VolatilityAdder      = CV_revenue_3m × 180 bps  [cap 125 bps]
        + RunwayAdder          = max(0, (8 − Weeks) × 12) [cap 120 bps]
        + StressAdder          = AdverseShortfall×18 + SevereShortfall×9 [cap 175 bps]

Guardrails: ±150 bps/month cap · 20 bps dead-band · 24.99% APR hard ceiling
            Cumulative 3-month cap: ±300 bps
```

### Asymmetric Hysteresis — Section A5/B1

The engine implements a **stateful, multi-cycle asymmetric hysteresis** rule for tier transitions to prevent pricing volatility from single-cycle score dips:

```
T=1 UPGRADE:   If raw score maps to higher tier → upgrade immediately.
               Clears any pending downgrade state.

T=2 DOWNGRADE: If raw score maps to lower tier:
               Cycle 1 → Enter PENDING state. Preserve activeTier and terms.
               Cycle 2 → If breach persists, finalise downgrade.
               Recovery → If score recovers before Cycle 2, cancel downgrade.
```

**Why this matters:** A borrower hit by a one-cycle inventory disruption keeps their pricing tier for one month. If they recover, no penalty is applied. Only sustained deterioration triggers repricing.

```javascript
// State vector per borrower
HST[id] = {
  activeTier,           // governs all pricing adders and routing
  pendingDowngrade,     // target tier if breach persists (or null)
  consecutiveMonthsBelow // 0, 1, or reset to 0 on finalisation
}
```

### Velocity Hierarchy (Adjustment Speed)

| Term | Cadence | Cap | Notice |
|---|---|---|---|
| Interest Rate | Monthly | ±150 bps/month | 10 business days |
| Credit Limit | Monthly | ±10% auto / ±20% senior | 15 days for reductions |
| Collateral | **Fixed at origination** | PPSA GSA — no automated adjustment | N/A |

### Revenue Bands

| Band | Revenue | Calibration | Facility Floor |
|---|---|---|---|
| `sub_150K_ineligible` | < $150K | Ineligible — blocked at Layer 1 | — |
| `band_150K_499K` | $150K–$499K | Sub-segment: DSCR +0.15 owner-comp normalisation | $15K |
| `band_500K_plus` | $500K–$5M | Standard | $25K |

### Routing Tiers

| Decision | Trigger Conditions |
|---|---|
| `AUTO_EXECUTE` | All changes within thresholds. 10bd notice to borrower. |
| `UNDERWRITER_REVIEW` | Rate Δ > 75 bps, limit Δ > 10%, or tier downgrade finalised |
| `SENIOR_ESCALATION` | Active tier = Red, score moved > 20 pts, coverage at Senior/Workout |
| `DRAW_FREEZE` | Runway < 4w with adverse shortfall, NSF ≥ 2, overdue > 30d, or AR 90+ > 50% in Red |

### Collateral / PPSA GSA Coverage Alerts

| Level | Coverage Ratio | Action |
|---|---|---|
| Healthy | > 150% | Internal log only |
| Monitor | 100–150% | Included in underwriter report |
| Escalate | 75–100% | Early warning — underwriter review |
| Senior | 50–75% | Cure notice process; legal team engaged |
| Workout | < 50% | Special Accounts; GSA enforcement evaluated |

> Collateral coverage is **never a borrower-facing term** — it is an internal monitoring signal only. The PPSA GSA is fixed at origination; re-registration is not an automated engine output.

### Human-AI Boundary

```
AUTO-EXECUTE:          Rate within ±150 bps band + limit ≤ 10% change
UNDERWRITER-REVIEW:    Rate near 3m cumulative cap + limit > 10%
SENIOR-ESCALATION:     Score < 35 (Red entry) + draw freeze confirmation
DRAW FREEZE:           Auto-trigger; Senior confirms within 24 hours
COVERAGE ENFORCEMENT:  Human + legal only — absolute boundary
CAPITAL DEPLOYMENT:    Human only — absolute boundary
```

---

## Canadian Regulatory Grounding

| Regulation | Constraint Applied |
|---|---|
| **PPSA (Personal Property Security Act)** | Single GSA at origination; no re-registration from engine outputs |
| **Criminal Code s.347** | Hard ceiling at 24.99% APR |
| **Cost of Borrowing Regulations** | Every rate change requires plain-language disclosure; `disclosure_text_hash` stored in audit log |
| **FSRA / non-bank scope** | Architecture designed for FSRA-regulated lenders to avoid OSFI B-20/Basel III conflicts |
| **PIPEDA** | Canadian data handling framework for Plaid/Flinks bank feed integration |

---

## Dashboard Features

### Three-Column Layout
- **Left panel** — Borrower portfolio list with active tier, score, hysteresis pending badge, revenue band
- **Center panel** — Full scrollable assessment dashboard per borrower
- **Right panel** — Live audit log with colour-coded entries by type; early warning indicator tags

### Center Dashboard Sections
1. **KPI Cards** — Cash balance, credit available, AR/AP ratio, weekly net cash flow
2. **Cash Forecast Chart** — 8-week historical + 6-week forward (Base/Adverse/Severe fan), Chart.js
3. **Stress Test Table** — Week 4 and Week 6 balances per scenario with PASS/MARGINAL/FAIL
4. **AI Triage Pipeline** — Animated 10-step processing pipeline
5. **Decision Card** — Routing outcome with reason and action bullets
6. **Score Bar + Hysteresis Banner** — Active tier vs raw tier, grace period status, breach count
7. **Term Cards** — Rate, limit, collateral coverage, next review — each with cadence tag
8. **Term History Table** — Last 6 months of score, tier, rate, limit, coverage, action
9. **AI-Generated Underwriting Memo** — Auto-written narrative per borrower per cycle

---

## File Structure

```
sme-credit-triage-engine/
├── index.html          # Self-contained simulation (all CSS + JS inline)
├── README.md           # This file
├── ARCHITECTURE.md     # Deep-dive specification: scoring, formulas, regulatory grounding
├── CHANGELOG.md        # Version history
├── CONTRIBUTING.md     # How to contribute
├── LICENSE             # MIT License
└── .github/
    └── ISSUE_TEMPLATE/
        ├── bug_report.md
        └── feature_request.md
```

---

## Deployment

### GitHub Pages (recommended)

1. Fork or clone this repository
2. Go to **Settings → Pages**
3. Set source to `main` branch, `/ (root)`
4. Visit `https://YOUR_USERNAME.github.io/sme-credit-triage-engine/`

### Local

```bash
git clone https://github.com/YOUR_USERNAME/sme-credit-triage-engine.git
cd sme-credit-triage-engine
open index.html    # macOS
# or
xdg-open index.html  # Linux
# or simply drag index.html into any browser
```

No build step, no `npm install`, no server required.

---

## Design Principles

**Explainability is non-negotiable.** The architecture is deliberately constrained to rule-based and lightweight model approaches so underwriters can understand and justify every decision. ML is noted as a future additive layer, not a replacement.

**Adjustment cadence reflects real-world constraints.** The speed hierarchy for term changes is rooted in operational, legal, and relationship-banking realities — not just technical preference.

**Human-AI boundaries must be explicit.** Routing tiers formalise when automation acts versus when humans must decide.

**Abuse prevention is a first-class design concern.** Any human-input mechanism requires corroboration requirements, frequency caps, and pattern detection baked in from the start.

---

## Limitations & Disclaimers

- This is a **simulation and design prototype**, not a production credit system
- Borrower data is synthetic; all financial figures are illustrative
- The engine has not been reviewed or approved by FSRA, OSFI, or legal counsel
- Regulatory classification of this instrument should be obtained before any real deployment
- ML enhancement path (XGBoost + SHAP values) is noted as a future layer; current implementation is rule-based only

---

## Roadmap

- [ ] Human-in-the-Loop Transaction Annotation module (DSCR_raw vs DSCR_adjusted)
- [ ] Plaid API integration layer (live bank feed replacement for synthetic data)
- [ ] NAICS code–adjusted industry scoring calibration
- [ ] Counterparty concentration risk module
- [ ] Covenant tripwire and seasonal adjustment overlays
- [ ] XGBoost + SHAP values ML enhancement layer (additive, not replacement)
- [ ] Annotation feedback loop for transaction classifier improvement

---

## License

MIT — see [LICENSE](LICENSE).

---

*Architecture designed for Ontario SME lenders operating in the gap between chartered bank rates (~6% APR) and private lender rates (10–18%+ APR). Approximately 430K–440K Ontario SMEs; roughly half seek external financing annually.*
