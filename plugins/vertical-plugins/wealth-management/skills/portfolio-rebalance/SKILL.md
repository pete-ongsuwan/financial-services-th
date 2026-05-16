---
name: portfolio-rebalance
description: Analyzes allocation drift across a multi-asset portfolio and generates rebalancing recommendations. Use this skill whenever the user asks about portfolio rebalancing, sleeve drift, allocation gaps, target deviation, tranche sizing, deployment plans, or asks "how am I doing vs my plan" / "quick AUM check" / "where are we underweight". Triggers on phrases like "rebalance", "drift", "vs target", "sleeve allocation", "AUM check", "underweight", "overweight", "redeploy", or any reference to the Target sheet in AUM.xlsx. Designed for a Thailand-based multi-asset portfolio in Thai Baht (THB) with five sleeves (Bond/Cash, Equity Funds, SET, Mixed, Alternative). Reads from AUM.xlsx in the active project folder. Does NOT apply US tax rules (no wash-sale logic, no tax-loss harvesting overlay) — those concepts do not map to the Thai retail mutual fund regime. In Thailand, capital gain from Thailand-based mutual funds and stocks selling is generally tax-exempt. However, dividend from both mutual funds and stocks is subject to a 10% withholding tax. Always denominate outputs in THB. For live data (prices, FX), prioritize project knowledge and reputable Thai financial sources; never invent values. Follow the structured output format for drift summary, actionable items, FX exposure, and open questions. Always mark data confidence (confirmed, estimated, requires user input). If AUM.xlsx is missing, ask the user to upload it before proceeding.
---

# Portfolio Rebalance (Thailand Localization)

## Purpose

Analyze the user's actual portfolio allocation against the Target sheet in `AUM.xlsx` and produce a rebalancing recommendation tailored to a Thailand-based, moderate-risk, growth-oriented retail investor with a long horizon (5+ years).

## Governing reference

The **Target sheet in `AUM.xlsx`** is the authoritative governing document. Separate outlook documents (e.g., `2026_Economic_Outlook_Investment_Plan_Revised.docx`, IMF/OECD/JPMorgan/BOT PDFs) are reference artifacts only — never override the Target sheet without explicit user confirmation. If the Target sheet has been revised, the latest version wins.

## Sleeve structure

The portfolio is organized into five sleeves. Always read current targets from the Target sheet in `AUM.xlsx` at runtime. The percentages below are fallback values only — used when the Target sheet is missing or a sleeve row cannot be parsed, and marked ⚠️ Estimated in output. As of last known revision:

| Sleeve | Target % | Notes |
|---|---|---|
| Bond/Cash | 32% | Also include brokerage cash or cash-equivalent assets |
| Mixed | 7% | Balanced funds |
| Equity Funds | 44% | Global core + thematic satellites |
| SET Portfolio | 10% | Thai direct equities |
| Alternative/Gold | 7% |

Effective equity exposure target: compute at runtime as (Equity Funds target % + SET target % + equity portion of Mixed sleeve target %). Do not use a hardcoded figure — it will become stale if targets change in AUM.xlsx.

## Inputs expected

| Input | Source | Format |
|---|---|---|
| Target allocations | `AUM.xlsx` → Target sheet | Sleeve name, target % |
| Policy rules | `AUM.xlsx` → Note sheet | Drift thresholds, concentration limits, tranche defaults |
| Current holdings | `AUM.xlsx` → Holdings sheet | Fund/stock name, sleeve, units, NAV/price, value (THB) |
| USD/THB rate | Web search if needed | Current spot, with source and timestamp |
| Current macro context | Project knowledge or web search | For trigger calibration |

If `AUM.xlsx` is not in the active project folder, ask the user to provide it before proceeding. Do not invent values.

## Workflow

1. **Read inputs.** Open `AUM.xlsx`. Identify Target, Holdings, and Note sheets. Note total AUM (THB).
   - Read sleeve target percentages from the Target sheet. If the Target sheet is missing or a sleeve target cannot be parsed, fall back to the percentages in the Sleeve structure table above and mark that sleeve's target as ⚠️ Estimated.
   - Read policy rules from the Note sheet (drift thresholds, concentration limits, tranche defaults, SET overweight resolution). If the Note sheet is missing or a rule cannot be parsed, fall back to the hardcoded values in the Key constraints section below and mark those rules as ⚠️ Estimated.
2. **Compute drift.** For each sleeve: `drift_% = actual_% − target_%`. For each holding: relative weight within its sleeve.
3. **Compute THB gap.** `gap_THB = drift_% × total_AUM` per sleeve.
4. **Flag actionable items.** Apply drift thresholds and concentration limits from the Note sheet (fallback: |drift| > 3% is actionable; any single SET holding > 25% of SET sleeve is a concentration flag).
5. **Propose deployment.** For each actionable sleeve:
   - Apply tranche structure from the Note sheet (fallback: 2 tranches of 50% each)
   - Apply staged deployment during uncertainty (geopolitical, macro shocks)
   - Use price-trigger + time-fallback pattern (e.g., target price level, with N-day fallback)
6. **Surface FX exposure.** Compute unhedged global equity THB value. Apply FX hedge logic:
   - Post-ceasefire / THB appreciation scenario → favor hedged instruments
   - Escalation / THB weakening scenario → favor unhedged
7. **Cross-check against standing triggers.** Check active project files and prior conversation context for any standing trigger notes (e.g., a specific stock trim level, stop-loss review, or gold tranche dip threshold). Note whether current state moves toward or away from those triggers.
8. **Output** as the structured format below.

## Key constraints

- **Denominate everything in THB.** Convert any USD figures using the latest USDTHB rate; cite source and timestamp.
- **Policy rules source:** read drift thresholds, concentration limits, tranche defaults, and SET overweight resolution from `AUM.xlsx` → Note sheet. The values below are fallback defaults, used only when the Note sheet is unavailable, and marked ⚠️ Estimated in output.
  - SET-Holdings overweight: dilute via growth of other sleeves, NOT by trimming SET positions across the board
  - SET stock cap: never propose a single SET stock exceeding 3% of total AUM; size proposals as % of SET sleeve (not total AUM)
  - Drift thresholds: ✅ on target (|drift| ≤ 1%), ⚠️ minor drift (1% < |drift| ≤ 3%), 🔴 actionable (|drift| > 3%)
  - Default tranche structure: 2 tranches of 50% each
- **Tax overlay (fixed — not in AUM.xlsx):** do NOT apply US wash-sale rules. Do NOT propose tax-loss harvesting as a structural strategy. Capital gain from Thailand-based mutual funds and stocks selling is generally tax-exempt. However, dividend from both mutual funds and stocks is subject to a 10% withholding tax.
- **Anti-pattern: covered-call / option-overlay ETFs (fixed — not in AUM.xlsx).** Do NOT propose products like WINC or similar yield-overlay funds. They structurally cap upside and conflict with the long-term capital growth mandate.

## Output format

Always produce these four sections in this exact order:

### 1. Drift summary table

```
| Sleeve         | Target % | Actual % | Drift % | Gap (THB)  | Status      |
|----------------|----------|----------|---------|------------|-------------|
| Bond/Cash      | [AUM.xlsx Target sheet] | xx.x | ±x.x | ±xxx,xxx | ✅/⚠️/🔴 |
| Mixed          | [AUM.xlsx Target sheet] |  x.x | ±x.x | ±xxx,xxx | ✅/⚠️/🔴 |
| Equity Funds   | [AUM.xlsx Target sheet] | xx.x | ±x.x | ±xxx,xxx | ✅/⚠️/🔴 |
| SET            | [AUM.xlsx Target sheet] | xx.x | ±x.x | ±xxx,xxx | ✅/⚠️/🔴 |
| Alternative    | [AUM.xlsx Target sheet] | xx.x | ±x.x | ±xxx,xxx | ✅/⚠️/🔴 |
```
_(Target % is read from `AUM.xlsx` → Target sheet. If unavailable, fallback values from the Sleeve structure table are used and marked ⚠️.)_

Status codes (thresholds read from `AUM.xlsx` → Note sheet; fallback values shown):
- ✅ on target (|drift| ≤ 1%)
- ⚠️ minor drift (1% < |drift| ≤ 3%)
- 🔴 actionable (|drift| > 3%)

### 2. Actionable items

Sorted by absolute drift, descending. For each:

```
**[Sleeve name]** ([±drift %], THB ±[gap])
- Rationale: why this drift matters now
- Proposed tranches: tranche 1 (THB amount, trigger), tranche 2 (THB amount, trigger)
- Time-based fallback: if triggers don't fire within N days, execute at market
- Confidence: ✅ confirmed / ⚠️ estimated / ❓ requires user input
```

### 3. FX exposure note

- Unhedged global equity exposure: THB xxx,xxx (x.x% of AUM)
- Current USDTHB: x.xx (source: ..., as of ...)
- Hedge tilt recommendation under current macro scenario: [hedged/unhedged/balanced]
- Standing FX-relevant triggers from active project files or prior conversation context (if any)

### 4. Open questions

List explicit decisions required from the user before execution. Examples:
- Should commodity sleeve initiation (SCBCOMP vs Principal GCF) proceed before reducing Bond/Cash further?
- Confirm gold tranche 3 trigger remains at XAUUSD $4,200–4,500?
- Any tactical view on USDTHB that should override the default hedge tilt?

## Data confidence convention

Mark every numeric data point in the output:
- ✅ Confirmed — read directly from AUM.xlsx
- ⚠️ Estimated — reconstructed, computed, or from a secondary source
- ❓ Requires user input — not available, ask before proceeding

Never present an estimated value without the ⚠️ marker. Never invent a confirmed value.

## Source priority for live data

When the skill needs current market data (prices, FX, commodity levels):

1. Project knowledge (uploaded reports: IMF, OECD, JPMorgan, BOT outlooks)
2. Web search — highest preferred sources to lowest:
   - https://www.innovestx.co.th/cafeinvest
   - settrade.com via Google search snippets (NEVER direct fetch — JavaScript-rendered, fails silently)
   - English: Reuters, Bloomberg, official central bank releases
   - Thai-language financial aggregators: Kaohoon, Mitihoon, Thunhoon (for SET data)
3. User confirmation — for sleeve-internal positions if Holdings sheet appears stale

## Example invocation

User: `/portfolio-rebalance`

Expected behavior:
1. Skill reads `AUM.xlsx` from the active project folder
2. Computes drift across all five sleeves
3. Identifies that Equity Funds is most underweight (illustrative: −4.9%, ~THB 179K gap)
4. Proposes two tranches of ~THB 90K into existing equity fund positions
5. Suggests USDTHB-based triggers and 30-day fallback
6. Flags Alternative sleeve overweight (Gold above target) and recommends pausing further gold tranches until drift narrows
7. Lists FX exposure and current hedge tilt recommendation
8. Asks 1–3 explicit open questions before any execution

## Localization notes

This skill replaces the upstream Anthropic `portfolio-rebalance` skill with Thailand-specific assumptions. Key deviations from upstream:

| Upstream assumption | Localized behavior |
|---|---|
| USD denomination | THB denomination |
| Stocks / bonds / cash structure | 5-sleeve structure (Bond/Cash, Equity Funds, SET, Mixed, Alternative) |
| US wash-sale rules | Removed — not applicable in Thai mutual fund regime |
| Tax-loss harvesting overlay | Removed — Thai tax treatment differs; out of scope |
| US tickers (AAPL, SPY, VTI) | Thai mutual fund codes + SET tickers |
| Generic advisor-to-client framing | Self-directed retail investor framing |
| Generic risk profile | Moderate-active, growth-oriented, 5+ year horizon |

If Anthropic updates the upstream skill, review their changes before merging. Preserve all of the constraints in the "Key constraints" section above.
