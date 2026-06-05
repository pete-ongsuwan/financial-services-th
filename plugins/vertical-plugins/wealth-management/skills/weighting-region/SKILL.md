---
name: weighting-region
description: Reads AUM.xlsx and produces a summarized table of portfolio assets aggregated by investing region, with region buckets defined per asset class. Use this skill whenever the user asks for regional allocation, geographic exposure, "weighting by region", "region breakdown", "how much is in Thailand vs Global", "EM exposure", "developed vs emerging split", or any per-region aggregation of holdings. Reads the Holdings sheet of AUM.xlsx (per-asset rows with a 'Region' column) and the Target sheet for asset-class membership. Region bucketing differs by asset class: BOND collapses to just Thailand / Global; EQUITY keeps each developed country/region separate, collapses developing/emerging/frontier markets into EM (but splits Thailand out separately even though it is technically EM), buckets global-mandate funds as Global, sector-mandate funds as Sector, and aggregates individual stocks to the market where they are listed (e.g. SET-listed stocks → Thailand). ALTERNATIVE buckets by what the asset is rather than region: mutual funds by their underlying (e.g. Gold Funds) and physical assets by what they are (e.g. Gold (Physical)). MIXED/balanced funds are dissolved (look-through) into BOND/EQUITY/ALTERNATIVE using a sub-allocation read from AUM.xlsx, or asked from the user, falling back to the Other-asset-classes rule if unresolved. CASH aggregates all cash and cash-equivalent assets by currency (ISO 4217), showing native amount and THB value, and sorts last in the combined table. "Domestic" always means Thailand. Designed for a Thailand-based multi-asset portfolio denominated in Thai Baht (THB). Output is a markdown table in chat. If AUM.xlsx is missing, ask the user to upload it before proceeding; never invent values.
---

# Weighting by Region

## Purpose

Read `AUM.xlsx` from the active project folder and produce a summarized / aggregated table of portfolio assets grouped by **investing region**, applying region-bucketing rules that differ by asset class. Output is denominated in Thai Baht (THB).

This skill is read-and-summarize only. It does not propose trades, drift, or rebalancing — for that, use `portfolio-rebalance`.

## Inputs expected

| Input | Source | Notes |
|---|---|---|
| Per-asset rows | `AUM.xlsx` → Holdings sheet | One row per fund/stock: name/code, asset class, **Region** column, value (THB) |
| Asset-class membership | `AUM.xlsx` → Target sheet | Main asset classes (BOND, EQUITY, …) live here; use to confirm each holding's class |
| Region | `AUM.xlsx` → Holdings sheet, **`Region`** column | Read directly. This is the authoritative region/mandate label per asset. |
| MIXED sub-allocation | `AUM.xlsx` (Holdings columns / Target breakdown / look-through sheet), else user | Per-fund BOND/EQUITY/ALTERNATIVE split for dissolving mixed-asset funds; see MIXED rules |
| CASH currency & amount | `AUM.xlsx` → Holdings sheet (currency field + native amount + THB value) | For CASH holdings, the ISO 4217 currency and native amount; THB value for conversion. See CASH rules |

If `AUM.xlsx` is not in the active project folder, ask the user to provide it before proceeding. Do not invent values. Always read the `Region` column directly — do not guess a region the column does not state. If a row's `Region` value is blank or unrecognized, list it under a **`Unclassified`** bucket and flag it for user input rather than forcing it into a bucket.

## Convention

- **Domestic = Thailand** in every context below.
- All values in **THB**, read from the Holdings sheet value column. Never convert or invent values; if a value is missing, flag it.
- Asset class is determined by the Target sheet's main asset classes (BOND, EQUITY, …). Match each Holdings row to its class. If a holding's class cannot be determined, flag it rather than assuming.

## Region-bucketing rules (differ by asset class)

### BOND

Only **two** buckets:

| Bucket | Rule |
|---|---|
| `Thailand` | `Region` column value is Thailand / Domestic |
| `Global` | Any other `Region` value |

Use the same `Region` column value as everything else: map Thailand/Domestic → `Thailand`, everything else → `Global`.

### EQUITY

Apply these rules **in order** — first match wins:

1. **Individual stocks** → aggregate to the **market country where the stock is listed**. The `Region` column should carry the listing market; e.g. all SET-listed stocks → `Thailand`. A stock listed on NYSE/NASDAQ → `United States`, LSE → `United Kingdom`, TSE → `Japan`, etc.
2. **Thailand** → always its own bucket `Thailand`, even though Thailand is technically an emerging market. This split-out takes precedence over the EM rule below.
3. **Developed countries/regions** → keep **each region/country separate**, using the region/country name from the `Region` column (e.g. `United States`, `Europe`, `Japan`, `Asia ex-Japan (Developed)`). Do not collapse developed markets together.
4. **Developing / Emerging Market / Frontier Market** funds → collapse together into a single `EM` bucket (Thailand already excluded by rule 2).
5. **Global-mandate mutual funds** (invest globally, no single-region tilt) → `Global`.
6. **Sector-mandate mutual funds** (invest in a specific sector rather than a region — e.g. technology, healthcare, energy) → `Sector`.

Notes:
- Rules are ordered so a Thai stock or Thai fund lands in `Thailand` (rules 1–2) before the EM rule (rule 4) can catch it.
- A fund whose `Region` names a single developed country/region is bucketed by that name (rule 3); a fund whose `Region` says global/world → `Global`; a fund whose `Region` names a sector → `Sector`; a fund whose `Region` says EM/emerging/developing/frontier → `EM`.
- If the `Region` column distinguishes "individual stock listing market" vs "fund mandate" only implicitly, prefer the literal `Region` value and surface any ambiguous rows in the open-questions note.

### ALTERNATIVE

ALTERNATIVE buckets by **what the asset is / what it holds**, not by investing region. Apply in order — first match wins:

1. **Mutual funds** (e.g. gold funds, commodity funds, REIT funds) → bucket by **what the fund invests in**, labelled `<Underlying> Funds` — e.g. a gold fund → `Gold Funds`, a broad commodity fund → `Commodity Funds`, a REIT fund → `REIT Funds`.
2. **Physical assets** → bucket by **what the asset is**, labelled `<Asset> (Physical)` — e.g. physical gold → `Gold (Physical)`, physical silver → `Silver (Physical)`.

Notes:
- Distinguish fund vs physical from the holding's name/type (a fund code or "fund/ETF" in the name → mutual fund; a bullion/bar/physical descriptor → physical). If a row is ambiguous, group by raw label and flag it.
- Keep a gold mutual fund (`Gold Funds`) and physical gold (`Gold (Physical)`) as **separate** buckets — do not merge them, since the user wants the fund-vs-physical distinction preserved.
- The `Region` column is not the bucket key for ALTERNATIVE; ignore it for bucketing (but you may surface it in the Notes if relevant).

### MIXED (mixed-asset / balanced funds)

MIXED-asset funds are **dissolved (look-through) into the main asset classes** — BOND, EQUITY, ALTERNATIVE — rather than shown as their own class. A mixed fund's THB value is split by its sub-allocation, and each piece is then re-bucketed under that main class's own rules above. Resolve the sub-allocation in this order:

1. **Sub-allocation declared in `AUM.xlsx`** → use it. Look for the fund's bond/equity/alternative split in the workbook — e.g. dedicated columns on the Holdings sheet (`% Bond` / `% Equity` / `% Alternative`, or a `Sub-allocation` field), a per-fund breakdown on the Target sheet, or a separate look-through/allocation sheet. Read the literal percentages; do not estimate them. If a region is also declared per sub-piece, use it for re-bucketing.
2. **Ask the user** → if no sub-allocation is declared anywhere in `AUM.xlsx`, ask the user for the fund's split: what percent goes to BOND, EQUITY, and ALTERNATIVE. State the assumption that, absent a declared region, the dissolved pieces are re-bucketed as **BOND → `Global`, EQUITY → `Global`, ALTERNATIVE → `Others`**. Percentages should sum to ~100%.
3. **Fall back** → if there is no declared sub-allocation **and** the user gives no reply, an invalid reply, or percentages that do not sum to ~100%, do **not** dissolve the fund. Treat MIXED under the **Other asset classes** rule below (its own per-class section, bucketed by the literal `Region` value) and flag in the Notes that look-through was skipped and why.

When dissolving (paths 1–2):
- Split THB value: `piece_value = fund_value × sub-allocation %` for each of BOND / EQUITY / ALTERNATIVE.
- Re-bucket each piece under the destination class's rules: a BOND piece → Thailand/Global; an EQUITY piece → the EQUITY region rules; an ALTERNATIVE piece → its asset bucket. Use the declared region per piece if present; otherwise the path-2 defaults (BOND→`Global`, EQUITY→`Global`, ALTERNATIVE→`Others`).
- The dissolved pieces flow into the existing BOND / EQUITY / ALTERNATIVE per-class tables and the combined table — there is **no** standalone MIXED row when look-through succeeds.
- In the Notes, list each dissolved fund, the split used, and the source (AUM.xlsx vs user-provided) so the look-through is auditable.

### CASH

All **cash and cash-equivalent** assets belong to the `CASH` class — e.g. bank/savings balances, brokerage cash, money-market funds, short-term deposits, and T-bills/near-cash instruments. CASH aggregates **by currency**, not by region:

- Bucket each cash holding by its **currency**, expressed as an **ISO 4217** code (e.g. `THB`, `USD`, `EUR`, `JPY`, `SGD`).
- For each currency, sum the native **amount** (in that currency's own units) and its **value in THB**. Native amount and THB value differ for non-THB currencies (THB value = amount × FX rate); for `THB`, amount and THB value are equal.
- Determine currency from the holding's currency field if present, else infer from the account/instrument and flag low-confidence guesses. If the THB value of a non-THB balance requires an FX conversion not present in `AUM.xlsx`, do not invent the rate — flag it as ❓ requires user input (or source a rate per the data-confidence convention) rather than guessing.
- CASH always sorts **last** in the combined table.

### Other asset classes

If the Target sheet defines asset classes beyond BOND, EQUITY, ALTERNATIVE, MIXED, and CASH, aggregate each by the literal `Region` column value and present them in their own per-class section. Do not invent bucketing rules the user has not specified; if a class's buckets are ambiguous, group by the raw `Region` value and flag for confirmation. (A MIXED fund that could not be dissolved falls here too — see the MIXED fall-back above.)

## Workflow

1. **Read inputs.** Open `AUM.xlsx`. Read the Target sheet to enumerate the main asset classes. Read the Holdings sheet rows (name/code, asset class, `Region`, value THB). Note total AUM (THB).
2. **Assign each holding to its asset class** using the Target sheet's classes.
3. **Dissolve MIXED funds (look-through).** For each MIXED-asset/balanced fund, resolve its BOND/EQUITY/ALTERNATIVE sub-allocation per the MIXED rules above: (a) declared in `AUM.xlsx`, else (b) ask the user, else (c) fall back to the Other-asset-classes rule without dissolving. When dissolved, split the fund's THB value into per-class pieces that feed the BOND/EQUITY/ALTERNATIVE buckets in the next step.
4. **Bucket each holding** (and each dissolved MIXED piece) per the asset-class-specific rules above. For BOND/EQUITY the bucket is an investing region; for ALTERNATIVE the bucket is what the asset is (fund underlying or physical asset). First-match-wins ordering applies for EQUITY and ALTERNATIVE.
5. **Aggregate** value (THB) by (asset class, bucket). Compute each bucket's % of its asset class and % of total AUM.
6. **Flag** any blank/unrecognized `Region`, missing value, undeterminable class, or skipped MIXED look-through under `Unclassified`/Notes and call it out.
7. **Output** the markdown tables below.

## Output format

Produce one summary table **per asset class** (rows sorted by % of AUM descending), then a combined table with one total row per asset class.

### Per-class tables

In every per-class table, **sort the bucket rows by `% of AUM` descending** (largest exposure first); the `TOTAL` row stays last. The example rows below are illustrative ordering only — re-sort by the actual values.

**BOND — by region**

```
| Region   | Value (THB) | % of BOND | % of AUM |
|----------|-------------|-----------|----------------|
| Thailand | xxx,xxx     | xx.x%     | xx.x%          |
| Global   | xxx,xxx     | xx.x%     | xx.x%          |
| TOTAL    | xxx,xxx     | 100.0%    | xx.x%          |
```

**EQUITY — by region**

```
| Region          | Value (THB) | % of EQUITY | % of AUM |
|-----------------|-------------|-------------|----------------|
| Thailand        | xxx,xxx     | xx.x%       | xx.x%          |
| United States   | xxx,xxx     | xx.x%       | xx.x%          |
| Europe          | xxx,xxx     | xx.x%       | xx.x%          |
| Japan           | xxx,xxx     | xx.x%       | xx.x%          |
| EM              | xxx,xxx     | xx.x%       | xx.x%          |
| Global          | xxx,xxx     | xx.x%       | xx.x%          |
| Sector          | xxx,xxx     | xx.x%       | xx.x%          |
| TOTAL           | xxx,xxx     | 100.0%      | xx.x%          |
```

_(Developed-market rows are dynamic — emit one row per developed country/region actually present in the Holdings sheet. Do not show buckets with zero holdings.)_

**ALTERNATIVE — by asset**

```
| Bucket          | Value (THB) | % of ALTERNATIVE | % of AUM |
|-----------------|-------------|------------------|----------------|
| Gold Funds      | xxx,xxx     | xx.x%            | xx.x%          |
| Gold (Physical) | xxx,xxx     | xx.x%            | xx.x%          |
| TOTAL           | xxx,xxx     | 100.0%           | xx.x%          |
```

_(Buckets are dynamic — `<Underlying> Funds` for mutual funds and `<Asset> (Physical)` for physical assets. Keep fund vs physical of the same underlying as separate rows. Do not show buckets with zero holdings.)_

**CASH — by currency**

```
| Currency | Amount   | Value (THB) | % of CASH | % of AUM |
|----------|----------|-------------|-----------|----------|
| THB      | x,xxx    | xxx,xxx     | xx.x%     | xx.x%    |
| USD      | x,xxx    | xxx,xxx     | xx.x%     | xx.x%    |
| TOTAL    |          | xxx,xxx     | 100.0%    | xx.x%    |
```

_(One row per currency, ISO 4217 code. `Amount` is the native amount in that currency; `Value (THB)` is the THB-converted value. Rows sorted by % of AUM descending; the TOTAL row's Amount is left blank since currencies don't sum. Do not show currencies with zero balance.)_

(Repeat for any additional asset classes present, grouped by raw `Region` value.)

### Combined table

One row per asset class — totals only, no region/bucket breakdown. Sort the asset-class rows in this fixed order: **BOND, EQUITY, ALTERNATIVE, then any other classes present, then CASH last**.

```
| Asset class | Value (THB) | % of AUM |
|-------------|-------------|----------|
| BOND        | xxx,xxx     | xx.x%    |
| EQUITY      | xxx,xxx     | xx.x%    |
| ALTERNATIVE | xxx,xxx     | xx.x%    |
| CASH        | xxx,xxx     | xx.x%    |
| TOTAL       | xxx,xxx     | 100.0%   |
```

After the tables, add a short **Notes** block:
- Total AUM (THB) and the value sum reconciliation (per-class totals should sum to total AUM; flag any gap).
- Any `Unclassified` rows (blank/unknown `Region`, missing value, or undeterminable class) and what input is needed.
- Any EQUITY rows whose bucket was ambiguous (e.g. a fund that could read as both regional and sector) and the rule applied.
- Any ALTERNATIVE rows where fund-vs-physical was ambiguous and the rule applied.
- For each MIXED fund dissolved by look-through: the fund, its BOND/EQUITY/ALTERNATIVE split, and the source of that split (AUM.xlsx vs user-provided). For any MIXED fund **not** dissolved: that look-through was skipped and why (no sub-allocation + no/invalid user reply).

## Data confidence convention

Mark each aggregate:
- ✅ Confirmed — summed directly from Holdings sheet values read from `AUM.xlsx`.
- ⚠️ Estimated — a value or class was reconstructed/inferred; explain how.
- ❓ Requires user input — blank/unrecognized `Region`, missing value, or undeterminable class.

Never present an estimated or unclassified value as confirmed. Never invent a value or a region the `Region` column does not state.

## Example invocation

User: `/weighting-region` or "show me my allocation by region"

Expected behavior:
1. Reads `AUM.xlsx` Holdings + Target sheets from the active project folder.
2. Buckets BOND into Thailand / Global only.
3. Buckets EQUITY: SET stocks → Thailand; developed markets kept separate (US, Europe, Japan, …); developing/EM/frontier collapsed to EM (Thailand split out); global funds → Global; sector funds → Sector.
4. Buckets ALTERNATIVE by asset: gold funds → Gold Funds, physical gold → Gold (Physical), kept separate.
5. Dissolves MIXED funds via look-through into BOND/EQUITY/ALTERNATIVE using the AUM.xlsx sub-allocation; if none, asks the user for the split; if still unresolved, treats the fund under the Other-asset-classes rule and notes the skip.
6. Prints per-class tables (rows sorted by % of AUM descending) and a combined per-asset-class total table (BOND, EQUITY, ALTERNATIVE, other classes, then CASH last) in THB.
7. Flags any unclassified rows and asks for confirmation.
