# Changelog

All notable changes to this project are documented in this file.  
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [2.0.1] — 2026-05-18

### Added
- **Asymmetric hysteresis state machine** (Section A5/B1) — T=1 upgrade / T=2 downgrade rule
  - `HST` state tracker object per borrower (`activeTier`, `pendingDowngrade`, `consecutiveMonthsBelow`)
  - `resolveTier(id, score)` function with full event taxonomy: `stable`, `upgrade`, `hysteresis_pending`, `hysteresis_cleared`, `downgrade`
  - Hysteresis grace period UI banner in assessment output section (active tier vs raw tier display)
  - `⏳ PENDING` badge on borrower cards when a downgrade is in first-breach state
  - Hysteresis event entries in audit log with breach count and target tier
  - `hyst_state` field appended to audit record
- **Audit log hysteresis entries** — specific coloured log entries for each transition event type
- **Score bar dual-tier display** — shows both `active tier` and `raw tier` when they diverge
- **Hysteresis status banners** — stable confirmation, grace period warning, upgrade-eligible notice

### Changed
- `cRate()` now accepts `activeTier` string directly instead of computing tier from raw score — pricing adders are anchored to hysteresis-governed tier
- `cRoute()` receives `activeTier` — Red-tier escalation checks use active tier, not raw score tier
- `render()` reads `HST[id].activeTier` for all UI labels; tier is not recomputed from score on display
- `runCycle()` commits state mutation via `resolveTier()` before computing rate/limit/routing
- Left panel borrower cards use `HST[id].activeTier` for border colour and tier pill
- Underwriting memo includes hysteresis context in Tier, Summary, and Terms sections
- `buildMemo()` updated with `activeTier`, `rawTier`, and `hst` parameters
- Step 7 overlay label updated: "TERMS — Rate + limit (anchored to active tier)"
- Audit record includes `hyst_state` field in final entry

### Fixed
- Rate adder was previously using a stateless score-to-tier lookup, which could penalise borrowers during grace periods with the wrong tier premium
- Routing was escalating on raw score tier = Red rather than active tier = Red

---

## [2.0.0] — 2026-05-17

### Added
- **Cash forecast chart** — 8-week historical + W0 anchor + 6-week Base/Adverse/Severe fan; Chart.js
  - All three forecast lines start at W0 (today), diverge forward only
  - Historical (cyan, solid) runs W−8 to W0 independently
  - W0 "Today" vertical marker with subtle dashed grid highlight
- **Stress test table** — Week 4 / Week 6 balances per scenario, PASS/MARGINAL/FAIL status
- **AI-generated underwriting memo** — auto-written narrative per borrower per cycle
- **Term history table** — last 6 months of score, tier, rate, limit, coverage, action
- **Score component breakdown bar** — Runway/DSCR/AR/Payment/Stress with colour-coded bars
- **4× KPI cards** — Cash balance, credit available, AR/AP ratio, weekly net CF
- **Early warning bar** — context-sensitive critical/warning message with field-specific triggers
- **Revenue band display** — shown in borrower cards, center header, and inputs panel
- **10-step animated pipeline** — icons + labels + live animation during `runCycle()`
- Sub-$150K ingestion block — assessment blocked at Layer 1 with dedicated UI state
- Sub-segment calibration note in assessment output when band = `band_150K_499K`
- **Three-column layout** — borrower list (left), scrollable dashboard (center), audit log (right)
- **Right panel audit log** — colour-coded entries by type (info/warn/alert/ok), entry count
- **Early warning indicator tags** — 9 indicators with critical/active/clear states
- Monthly/Quarterly cycle toggle — quarterly disables limit assessment

### Changed
- **Collateral term card** redesigned as internal-only monitoring (removed borrower-facing dial)
  - Shows PPSA GSA fixed status + 5-level coverage alert (Healthy/Monitor/Escalate/Senior/Workout)
  - Speed tag changed from `Quarterly` to `Internal Only`
- **Velocity hierarchy finalised**: Rate = monthly · Limit = monthly · Collateral = fixed at origination
- Rate dead-band changed from 15 bps to 20 bps
- Monthly cap set at ±150 bps (was ±100 bps in v1.x)
- EWMA α fixed at 0.5, single monthly cadence (removed weekly α variant)
- Tier premium table updated: Green A 100bps, Green B 175bps, Yellow A 325bps, Yellow B 500bps, Red 800bps
- Base rate component: 550 bps (BoC Overnight + 150 bps structural spread)
- Pipeline expanded from 9 to 10 layers (added Coverage as L6, renamed Log → Audit as L10)
- `cRoute()` condition for Senior Escalation changed from `score < 35` to `activeTier === Red`
- Audit entries now include `disclosure_text_hash` and `input_data_hash`

### Removed
- Weekly cycle option (replaced by monthly-only with quarterly as an alternative)
- Collateral ratio as a borrower-facing adjustable term
- Weekly-specific EWMA alpha variant

---

## [1.0.3] — 2026-05-14 *(initial simulation)*

### Added
- Base simulation with 3 borrowers (Maple Brew, Nova Freight, Strata Build)
- 9-step animated processing pipeline
- Three-term output: Rate (weekly) · Limit (monthly) · Collateral ratio (quarterly)
- Routing decision card (Auto-Execute / Underwriter Review / Senior Escalation / Draw Freeze)
- Audit log with right panel
- Early warning indicator tags
- Inputs/sliders tab for scenario simulation
- Score gauge (canvas arc) with component bars
- Stress shortfall cards (3 scenarios)
- Dark theme: IBM Plex Mono/Sans Condensed, #0a0e14 background, #00d4ff accent, grid overlay
- Clipped-corner run button with shimmer hover effect
