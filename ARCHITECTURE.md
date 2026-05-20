# Architecture Specification
## SME Credit Triage Engine — CU/DT v2.0

> Deep-dive technical and regulatory specification. For the high-level overview see [README.md](README.md).

---

## Table of Contents

1. [System Context](#1-system-context)
2. [Revenue Band Classification](#2-revenue-band-classification)
3. [Scoring Engine](#3-scoring-engine)
4. [Asymmetric Hysteresis (A5/B1)](#4-asymmetric-hysteresis-a5b1)
5. [Interest Rate Formula](#5-interest-rate-formula)
6. [Credit Limit Logic](#6-credit-limit-logic)
7. [Collateral & PPSA Coverage Monitoring](#7-collateral--ppsa-coverage-monitoring)
8. [Stress Testing](#8-stress-testing)
9. [Routing & Human-AI Boundary](#9-routing--human-ai-boundary)
10. [Audit Log Schema](#10-audit-log-schema)
11. [Regulatory Grounding](#11-regulatory-grounding)
12. [Data Sources](#12-data-sources)
13. [Future ML Enhancement Path](#13-future-ml-enhancement-path)

---

## 1. System Context

The engine occupies the gap between chartered bank credit (typically ~6% APR, 2–8 week decisions) and private lending (10–18%+ APR, fast but expensive). It targets Ontario SMEs with $150K–$5M annual revenue who have a demonstrable cashflow history but insufficient collateral or operating history for traditional bank approval.

**Deployer type:** FSRA-regulated non-bank lender or Ontario credit union  
**Not designed for:** OSFI-regulated Schedule I/II banks (B-20/Basel III conflicts)  
**Origination gate:** Minimum score of Green B (≥ 65) at origination — the engine manages drift from a sound starting point

---

## 2. Revenue Band Classification

Revenue band is evaluated at ingestion (Layer 1) and determines eligibility, calibration, and facility floors.

```
Revenue < $150K      → sub_150K_ineligible    Blocked at Layer 1. Assessment does not proceed.
$150K ≤ Revenue < $500K → band_150K_499K     Sub-segment calibration active.
$500K ≤ Revenue ≤ $5M   → band_500K_plus     Standard calibration.
```

### Sub-segment Calibration ($150K–$499K)

Owner-operated businesses in this band frequently commingle personal and business cash flows. The DSCR metric systematically understates debt service capacity because owner draw is treated as an expense rather than discretionary compensation.

**Adjustment:** DSCR receives a +0.15 normalisation before DSCRScore computation. This is applied only within the scoring formula and is not reflected in the raw DSCR figure shown to the borrower.

**Facility floor:** $15,000 CAD (vs $25,000 for standard band)

---

## 3. Scoring Engine

### 3.1 Component Scores

All component scores are bounded [0, 100].

```
RunwayScore       = min(100,  (WeeksRunway / 12) × 100)
DSCRScore         = min(100,  (adjDSCR / 2.0) × 100)
                    where adjDSCR = DSCR + 0.15 if band_150K_499K, else DSCR
ARHealthScore     = max(0, 100 − (AR_90plus × 2) − max(0, AR_90plus − 10))
PaymentScore      = max(0, 100 − (DaysOverdue × 3) − (NSF_Events × 10))
StressShortfallScore = max(0, 100 − (SevereShortfall_weeks × 10))
```

### 3.2 Composite Raw Score

```
raw = 0.25 × RunwayScore
    + 0.25 × DSCRScore
    + 0.20 × ARHealthScore
    + 0.15 × PaymentScore
    + 0.15 × StressShortfallScore
```

### 3.3 EWMA Smoothing

```
smooth(t) = α × raw(t) + (1 − α) × smooth(t−1)
α = 0.5 (single monthly cadence)
```

EWMA dampens single-cycle volatility while remaining responsive to sustained trends. The α = 0.5 value weights current and prior period equally, providing a one-cycle memory.

### 3.4 Risk Tiers

| Tier | Smooth Score | PD Proxy | Tier Premium (bps) |
|---|---|---|---|
| Green A | ≥ 80 | < 1% | 100 |
| Green B | 65–79 | 1–3% | 175 |
| Yellow A | 50–64 | 3–8% | 325 |
| Yellow B | 35–49 | 8–18% | 500 |
| Red | < 35 | 18%+ | 800 |

---

## 4. Asymmetric Hysteresis (A5/B1)

### Rationale

Without hysteresis, a one-cycle score dip caused by a temporary inventory bottleneck, delayed AR collection, or seasonal cash trough would immediately reprice the borrower to a higher tier. This creates a negative feedback loop: higher rates reduce cash flow, which further stresses the score, which further raises rates.

The T=2 downgrade rule absorbs single-cycle volatility shocks while the T=1 upgrade rule ensures that genuine improvement is immediately rewarded.

### State Vector

```javascript
HST[borrower_id] = {
  activeTier:            string,  // current pricing-governing tier
  pendingDowngrade:      string | null,  // target tier if breach persists
  consecutiveMonthsBelow: number  // 0 | 1 | resets to 0 on finalisation
}
```

### Transition Logic

```
INPUT:  computedScore (EWMA smooth score this cycle)
OUTPUT: activeTier (tier that governs pricing and routing this cycle)

rawTier = staticLookup(computedScore)
currentRank = TIER_RANK[activeTier]
rawRank = TIER_RANK[rawTier]

IF rawRank > currentRank:
  // T=1 UPGRADE — immediate
  activeTier ← rawTier
  pendingDowngrade ← null
  consecutiveMonthsBelow ← 0
  emit event: 'upgrade'

ELSE IF rawRank < currentRank:
  IF pendingDowngrade != null AND TIER_RANK[pendingDowngrade] >= rawRank:
    // Second consecutive breach — finalise
    consecutiveMonthsBelow += 1
    IF consecutiveMonthsBelow >= 2:
      activeTier ← rawTier
      pendingDowngrade ← null
      consecutiveMonthsBelow ← 0
      emit event: 'downgrade'
    ELSE:
      emit event: 'hysteresis_pending'
  ELSE:
    // First breach — enter grace period
    pendingDowngrade ← rawTier
    consecutiveMonthsBelow ← 1
    emit event: 'hysteresis_pending'
  // activeTier unchanged — terms preserved

ELSE:
  // Score in range of active tier — stable
  IF pendingDowngrade != null:
    // Score recovered — cancel pending downgrade
    pendingDowngrade ← null
    consecutiveMonthsBelow ← 0
    emit event: 'hysteresis_cleared'
  ELSE:
    emit event: 'stable'
```

### Hysteresis Events and Audit Trail

Every cycle emits one of five events, each producing a specific audit log entry:

| Event | Meaning | Log Colour |
|---|---|---|
| `stable` | Score corroborates active tier | — (no special entry) |
| `upgrade` | T=1 immediate upgrade executed | Green |
| `hysteresis_pending` | First breach; grace period starts | Yellow |
| `hysteresis_cleared` | Score recovered; pending downgrade cancelled | Green |
| `downgrade` | T=2 rule satisfied; tier finalised | Red |

### Important: activeTier Propagation

The `activeTier` from HST — **not** the raw score tier — must propagate to:
- `cRate()`: tier premium adder selection
- `cRoute()`: Red-tier escalation check
- All UI tier labels and borrower card colours
- Underwriting memo tier reference
- Audit log tier field

---

## 5. Interest Rate Formula

```
Rate(t) = BaseRate + TierPremium + VolatilityAdder + RunwayAdder + StressAdder

BaseRate       = 550 bps  (BoC Overnight + 150 bps structural spread)
TierPremium    = f(activeTier)  [100 / 175 / 325 / 500 / 800 bps]
VolatilityAdder = min(125, round(CV_revenue_3m × 180))
RunwayAdder    = min(120, max(0, round((8 − WeeksRunway) × 12)))
StressAdder    = min(175, round(AdverseShortfall_weeks × 18 + SevereShortfall_weeks × 9))
```

### Guardrails

```
Monthly change cap:      ±150 bps  (symmetric — decreases fire as fast as increases)
Dead-band:               ±20 bps   (no change if proposed delta is within 20 bps)
Cumulative 3-month cap:  ±300 bps
Hard floor:              BaseRate + 200 bps = 750 bps
Hard ceiling:            2,499 bps = 24.99% APR (Criminal Code s.347)
Notice period:           10 business days
```

### Rate Symmetry Rule

Rate decreases must fire in the same cycle as score improvement. This is a non-negotiable design constraint: it ensures the borrower's financial discipline is rewarded immediately, not deferred.

---

## 6. Credit Limit Logic

### Monthly Assessment

```
IF score ≥ 73 AND dscr > 1.5 AND runway ≥ 10w AND utilisation > 60%:
  proposed_pct = +8%   (increase eligible)
ELSE IF score_drop ≥ 5 AND score_drop < 12:
  proposed_pct = −7%
ELSE IF score_drop ≥ 12 OR runway < 6w:
  proposed_pct = −15%
ELSE:
  proposed_pct = 0%

guardrailed_pct = max(−20%, min(+10%, proposed_pct))
new_limit = max(floor, round(currentLimit × (1 + guardrailed_pct)))
```

### Human Approval Thresholds

```
Auto-execute:        |Δ| ≤ 10%
Senior approval:     |Δ| > 10%  OR  any increase > 10%
```

### Notice Period

15 calendar days for reductions. No advance notice required for increases.

### Quarterly Cycle

In quarterly mode, credit limits are not assessed. Only interest rate is reviewed.

---

## 7. Collateral & PPSA Coverage Monitoring

### PPSA GSA — Fixed at Origination

The Personal Property Security Act General Security Agreement is a legal instrument registered at origination. The engine **never outputs a collateral adjustment**. Re-registration is prohibitively expensive for quarterly automated changes and requires legal process.

### Coverage Ratio Alerts (Internal Only)

```
Coverage Ratio = CollateralValue / OutstandingFacility

> 150%    → Healthy   — Internal log only; no action
100–150%  → Monitor   — Included in monthly underwriter report
75–100%   → Escalate  — Early warning; underwriter review
50–75%    → Senior    — Cure notice process; legal team engaged
< 50%     → Workout   — Special Accounts; GSA enforcement evaluated
```

The coverage ratio is an **internal monitoring signal** only. It is not disclosed to the borrower as a dynamic term. Enforcement action requires human decision-maker + legal team; the engine cannot initiate enforcement.

---

## 8. Stress Testing

Three forward-looking cash balance scenarios computed from current balance and weekly net cash flow:

```
Base:    Revenue ×1.0, Expenses ×1.0  (status quo continuation)
Adverse: Revenue ×0.65, weekly drift  (approx. −20% revenue impact + expense drag)
Severe:  Revenue ×0.35 − $4,000/week  (approx. −40% revenue + structural expense increase)
```

### Shortfall Computation

```
BaseShortfall    = max(0, 8 − runway)
AdverseShortfall = max(0, 8 − runway × 0.7 + 0.5)
SevereShortfall  = max(0, 8 − runway × 0.5 + 1.5)
```

Shortfall is measured in weeks below the 8-week threshold. A shortfall of 0 means the borrower survives the scenario above the minimum runway threshold.

### Stress PASS/MARGINAL/FAIL Thresholds

```
PASS:     Wk-6 balance > $120,000  (1.5× the $80K floor)
MARGINAL: Wk-6 balance $80,000–$120,000
FAIL:     Wk-6 balance < $80,000
```

---

## 9. Routing & Human-AI Boundary

### Decision Authority Matrix

```
DRAW_FREEZE (auto-trigger, Senior confirms within 24h):
  - runway < 4w AND adverse shortfall > 0
  - NSF events ≥ 2 in trailing 30 days
  - Days overdue > 30
  - AR 90+ > 50% AND active tier = Red

SENIOR_ESCALATION:
  - activeTier = Red
  - |score(t) − score(t−1)| > 20 pts
  - Coverage alert at Senior or Workout level

UNDERWRITER_REVIEW:
  - Rate Δ > 75 bps
  - |Limit Δ%| > 10%
  - Active tier downgrade finalised this cycle (T=2 satisfied)

AUTO_EXECUTE:
  - All other cases
  - 10 business day notice to borrower
  - Post-hoc review by underwriter
```

### Absolute Boundaries (Engine Cannot Cross)

```
CAPITAL DEPLOYMENT    → Human only; engine never allocates funds
GSA ENFORCEMENT       → Human + legal only; engine issues alert, not action
COVENANT CURE NOTICE  → Human signs; engine flags
ACCOUNT WRITE-OFF     → Human only
```

---

## 10. Audit Log Schema

Every cycle produces an immutable 41-field audit entry. Key fields:

| Field | Type | Description |
|---|---|---|
| `log_id` | UUID | Unique entry identifier |
| `borrower_id` | string | Borrower reference |
| `cycle_type` | enum | `monthly_risk_assessment` or `quarterly_rate_only` |
| `revenue_band` | enum | `sub_150K_ineligible` / `band_150K_499K` / `band_500K_plus` |
| `raw_score` | int | Unsmoothed composite score |
| `smooth_score` | int | EWMA-smoothed score (α=0.5) |
| `active_tier` | enum | Hysteresis-governed tier |
| `raw_tier` | enum | Stateless score-mapped tier |
| `hysteresis_event` | enum | `stable` / `upgrade` / `hysteresis_pending` / `hysteresis_cleared` / `downgrade` |
| `consecutive_months_below` | int | Breach count (0 when not pending) |
| `pending_downgrade_tier` | enum / null | Target tier if breach persists |
| `proposed_rate_bps` | int | Pre-guardrail rate |
| `guardrailed_rate_bps` | int | Post-guardrail rate |
| `rate_delta_bps` | int | Net change this cycle |
| `tier_premium_bps` | int | activeTier adder applied |
| `volatility_adder_bps` | int | CV-derived adder |
| `runway_adder_bps` | int | Runway-derived adder |
| `stress_adder_bps` | int | Stress-derived adder |
| `proposed_limit` | int | Pre-guardrail limit |
| `guardrailed_limit` | int | Post-guardrail limit |
| `coverage_ratio_pct` | int | Collateral coverage (internal) |
| `coverage_alert_level` | enum | Healthy / Monitor / Escalate / Senior / Workout |
| `routing_decision` | enum | AUTO_EXECUTE / UNDERWRITER_REVIEW / SENIOR_ESCALATION / DRAW_FREEZE |
| `disclosure_text_hash` | hex | Hash of plain-language change disclosure |
| `input_data_hash` | hex | Hash of input data snapshot |
| `model_version` | string | Engine version |
| `created_at` | ISO 8601 | Timestamp UTC |

---

## 11. Regulatory Grounding

### PPSA (Personal Property Security Act — Ontario)

- GSA registered at origination; never re-registered as an engine output
- Part V enforcement process is human-initiated; engine issues alert only
- Coverage ratio is an internal monitoring metric, not a disclosed term

### Criminal Code s.347

- Hard ceiling: 24.99% APR on all facilities
- Engine enforces this as an absolute guardrail; no override path exists

### Cost of Borrowing Regulations (Federal)

- Every rate change requires a plain-language disclosure to the borrower
- `disclosure_text_hash` stored in the audit log for every rate change
- 10 business day advance notice period before any rate change takes effect

### PIPEDA (Personal Information Protection and Electronic Documents Act)

- Governs data handling for bank feed integration (Plaid/Flinks)
- Consent scope must cover transaction-level data retrieval
- Data minimisation: only fields required for score computation are retained

### BDC Gap / Market Positioning

- Targets businesses too risky for chartered bank rates but too disciplined for private lenders
- Not designed to compete with BDC (serves higher-risk government mandate)
- Rate corridor: ~7–15% APR depending on tier (between bank ~6% and private 10–18%+)

---

## 12. Data Sources

### Primary Inputs

| Source | Data | Layer |
|---|---|---|
| Accounting CSV (QuickBooks / Xero) | Revenue, expenses, AR aging, AP | L1 Ingest |
| Bank feed — Plaid (CA) / Flinks (fallback) | Verified transactions, balances | L1 Ingest |
| Facility ledger | Outstanding balance, utilisation | L1 Ingest |
| Bank of Canada | Overnight rate | L1 Ingest |

### Derived Metrics

| Metric | Source | Notes |
|---|---|---|
| Cash runway (weeks) | Bank feed balance ÷ avg weekly burn | 8-week lookback |
| DSCR | Accounting: EBITDA ÷ debt service | LTM basis |
| AR 90+ days | Accounting: AR aging bucket | % of total AR |
| Revenue CV | Accounting: weekly revenue | Trailing 12 weeks |
| Days overdue | Facility ledger | |
| NSF events | Bank feed | Trailing 30 days |

---

## 13. Future ML Enhancement Path

The current architecture is deliberately rule-based and explainable. An ML enhancement layer is noted as a future additive path — it does not replace the rule-based engine.

### Proposed: XGBoost + SHAP Values

**Pre-condition:** Minimum 500 labelled loan outcomes (default / no-default) across the portfolio.

**Role:** The ML model produces a PD estimate that supplements (not replaces) the composite score. It is used as an additional input to the StressShortfallScore component or as a routing confidence modifier.

**Constraints:**
- All existing guardrails remain unchanged
- Human approval thresholds remain unchanged
- SHAP values are surfaced in the audit log for every ML-influenced decision
- The rule-based score continues to be computed independently for comparison

### Annotation Feedback Loop

The Human-in-the-Loop Transaction Annotation module (planned) allows borrowers to flag and explain misclassified transactions that distort DSCR. Accepted/rejected annotations accumulate as labelled ground truth to improve the transaction classifier's confidence scoring over time.

```
Raw DSCR    → displayed as DSCR_raw
Adjusted DSCR → DSCR + accepted annotations → DSCR_adjusted
Both values stored in audit log; underwriter adjudication required for large adjustments
```

---

*Specification version: 2.0 | Last updated: May 2026*
