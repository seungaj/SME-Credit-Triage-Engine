# Contributing

Thank you for your interest in contributing. This project is an architectural simulation of a credit triage engine — contributions that improve the accuracy of the engine logic, the realism of the simulation, or the clarity of the documentation are especially welcome.

---

## What to Contribute

### High-Value Contributions

- **Canadian regulatory accuracy** — corrections or additions to PPSA, Cost of Borrowing Regulations, PIPEDA, or Criminal Code s.347 constraints
- **Engine logic improvements** — scoring formula calibration, hysteresis parameter tuning, stress scenario refinements
- **Plaid/Flinks integration layer** — replacing synthetic data generation with actual bank feed API calls
- **NAICS code industry adjustments** — seasonal and sector-specific score calibration overlays
- **Accessibility improvements** — keyboard navigation, ARIA labels, screen reader support
- **Additional borrower archetypes** — new pre-loaded scenarios representing edge cases

### Out of Scope

- Introducing a build system, bundler, or framework dependency — the project is intentionally a single self-contained HTML file
- Any change that requires a backend server to function
- Real borrower data of any kind

---

## Getting Started

1. **Fork** the repository and clone your fork locally
2. Open `index.html` in a browser — no build step required
3. Make your changes directly to `index.html`
4. Test across Chrome, Firefox, and Safari
5. Submit a pull request with a clear description

---

## Code Style

The codebase uses minified-style JavaScript in the `<script>` tag for conciseness, but new engine logic functions should be written readably:

```javascript
// Good — readable function with clear purpose and parameters
function resolveTier(id, score) {
  const st = HST[id];
  const rawTier = rawTierFromScore(score);
  // ...
}

// Acceptable for UI helpers — brief lambdas
const fmt = n => new Intl.NumberFormat('en-CA', { ... }).format(n);
```

**CSS** — new styles go in the `<style>` block, following the existing section comment pattern (`/* ── SECTION NAME ── */`).

**Comments** — engine logic functions should have JSDoc-style comments explaining:
- What the function computes
- What side effects it has (particularly state mutations on `HST`)
- Which regulatory constraint or architecture section it implements

---

## Architecture Constraints

Before proposing changes to the engine logic, please read [ARCHITECTURE.md](ARCHITECTURE.md). Key constraints that must not be violated:

1. **`activeTier` (from `HST`) must govern all pricing adders** — never use raw score tier for rate computation
2. **Collateral is never a borrower-facing adjustable term** — coverage ratio monitoring is internal only
3. **PPSA boundary is absolute** — no automated output can trigger GSA re-registration or enforcement
4. **Criminal Code s.347 ceiling is a hard guardrail** — 24.99% APR maximum, no override path
5. **Rate decreases must fire in the same cycle as score improvements** (symmetry rule)
6. **State mutations on `HST` happen only inside `runCycle()`** — `render()` is read-only

---

## Submitting Issues

Please use the issue templates:

- [Bug Report](.github/ISSUE_TEMPLATE/bug_report.md) — for engine logic errors, UI bugs, or rendering issues
- [Feature Request](.github/ISSUE_TEMPLATE/feature_request.md) — for new capabilities or improvements

When reporting an engine logic bug, please include:
- The borrower state (input values) that triggered the issue
- The expected output and the actual output
- Which layer or function you believe contains the error

---

## Disclaimer

This is a simulation and design prototype. Contributors should not introduce real borrower data, real API keys, or production system integrations into the public repository.
