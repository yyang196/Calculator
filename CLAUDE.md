# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Retirement planning calculator used by **Yang Yang (GEFA senior financial consultant)** with clients. Two deliverables, each a **single self-contained HTML file** — no build, no backend, no test framework, no package manager. All logic runs in the browser.

1. **Desktop calculator** (`retirement.html`) — used in client-facing demonstrations.
2. **Mobile retirement wizard** (`retirement_wizard*.html`) — step-by-step pages clients open on their own phone, tablet or computer; ends with the adviser's contact card.

## File layout (what matters)

- `retirement.html` — the desktop calculator: inline `<style>`, inline JS, Chart.js v4 via CDN, Google Fonts. Edit this file directly; there is nothing else to compile.
- `retirement_wizard_v2.html` — **current wizard** (redesign; system fonts, responsive: phone < 640px, tablet 640–899px, two-column ≥ 900px). Charts are hand-drawn inline SVG (no Chart.js). Profile photo and WeChat QR are embedded as base64 (`AVATAR`, `QR`), so `profile.jpg` / `wechat_qr.jpeg` are source images only.
- `retirement_wizard.html` — wizard v1, kept for reference. Same calculation as v2.
- `deploy/index.html` — copy of `retirement_wizard_v2.html` that is uploaded to Cloudflare Pages (https://retirement-planner-sg.pages.dev). Refresh the copy and redeploy after changing v2; the copy does not update itself.
- `design-demos/` — design directions and spec for the wizard v2 redesign (historical reference).
- `docs/superpowers/specs/2026-05-25-retirement-calculator-design.md` — **authoritative design spec.** Contains the full calculation model (§7), KPI verification numbers (§10.4), and TVM mode invariants (§10.5). Update this whenever behavior changes.
- `Proposal_retire_invest.xlsx` — Excel reference (`UT-FI-10` worksheet) used to validate the Capital Drawdown recursion. Do not modify.
- `step*.png` — UI iteration screenshots (historical reference only).

## Running locally

```bash
python3 -m http.server 8765
# then open http://localhost:8765/retirement.html
```

The Playwright MCP browser tool is pre-permitted in `.claude/settings.local.json` for headless UI verification (screenshots, console checks, click-through testing).

## Calculation model — invariants you must preserve

The whole calculator is built around three TVM phases. Two non-obvious rules:

1. **Both accumulation and decumulation use BGN mode (annuity due).** This is deliberate — it matches the real cash-flow timing of insurance premiums / RSP / month-start income needs.
   - Accumulation: `b = (b + PMT) × (1 + m)` (add first, grow second) and the closed-form factor is `((1+m)^M − 1)/m × (1+m)`.
   - Decumulation: `B_t = (B_{t−1} − W_t) × (1 + r_post)` (withdraw first, grow second); LS closed form includes the `(1 + r_post)` annuity-due factor.
   - Cross-validate any change by ensuring `timeline[ageRetire].balance ≈ LS` and `timeline[ageEnd].balance ≈ 0`.

2. **Waiting Period is a sub-phase of pre-retirement.** It shares Phase I's `r_pre` and `i_pre` (no separate inputs). Contributions stop during waiting; capital still compounds. Index logic uses half-open intervals `[ageNow, ageContributionEnd)` and `[ageRetire, ageRetire+N)` so contribution-end and retirement-start never overlap visually or numerically, even when `yearsWait = 0`.

3. **Gender-driven default for `yearsRetired`:** Male → 20, Female → 25. Switching gender overwrites any manual adjustment (Singapore life-expectancy convention: 82M / 86F).

4. **Chapter 2 charts are decoupled from Chapter 3 strategies.** Chapter 2 always uses *planned* expense (capital reaches exactly 0 at end of plan). Chapter 3's CD withdrawal-rate slider must not affect Chapter 2 KPIs or charts.

## State & rendering pattern

Single global `state` object (see top of script block). Any UI input → mutate `state[key]` → call `render()`. `render()` calls `calculate()` (pure function returning all derived values) then updates DOM nodes by ID and re-instantiates the four Chart.js instances. There is no virtual DOM, no reactivity framework, no debouncing.

Sliders are wired by `setupSlider()` which reads `data-slider`, `data-min`, `data-max`, `data-step`. When code needs to update a slider value externally (e.g. gender switch → yearsRetired), it must manually update the slider's `.slider-fill` width, `.slider-thumb` position, and `.slider-val` text — there is no shared `updateUI()` accessible outside the closure. See the `ageNow → ageRetire` and `gender → yearsRetired` patterns in `init()`.

## Validation (no test framework)

Numeric correctness is validated two ways:

1. **Inline Node.js scripts** that replicate `calculate()`. Used for sanity-checking formula changes. Example pattern: re-implement the formulas in `node -e "..."`, then compare against the design doc's §10.4 KPI table.
2. **Visual verification** via Playwright MCP — drive the actual `retirement.html` in a browser, screenshot the rendered KPI cards and charts.

Whenever you change a calculation, update the design doc's §10.4 verification numbers in lockstep.

## Editing conventions

- Match the existing terse comment style — comments mark *why* (e.g. "BGN mode (add first, grow second)") not *what*.
- Currency formatting: `fmtMoney()` returns `en-SG`-locale strings; `parseMoney()` strips non-digits.
- Color tokens are CSS custom properties at the top of `<style>` (`--pre`, `--wait`, `--post`, `--cp`, `--cd`, etc.). Do not introduce ad-hoc hex values.
- Print/PDF output is driven by `@media print` rules; preserve them when editing visible UI.

## Communication preferences for this project

Yang Yang is the practitioner-user. Write directly to him in dialogue. He is technical enough to read the code but values: explicit assumption-surfacing, numerical verification before claims of correctness, and tight surgical changes over speculative refactoring. See the global `~/.claude/CLAUDE.md` for the full preference profile.
