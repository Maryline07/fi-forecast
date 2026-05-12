# Spec — «Financial Independence Forecast»

A single-page calculator for a financial-independence forecast: the user enters their current finances and a target, the page projects three scenarios (conservative / realistic / aggressive), and surfaces the year in which passive income covers the target.

## 0. Context

A small learning project in the `etape04` series. Sibling projects (`passive-income`, `portfolio-journal`, `compound`) all follow the same shape — one static HTML file with inline `<style>` and `<script>`, Chart.js via CDN, no build step. This spec applies the same shape to a new task: instead of reading from a CSV, the data comes from a form, and the artifact is an interactive page rather than a report.

## 1. Goal

Deliver `index.html` that — given the form inputs — builds **three scenario projections of capital over an H-year horizon** and answers one question: «when does my passive income cover my target, measured in today's purchasing power?»

Output blocks:
1. **Three "capital by year" tables** — one per scenario.
2. **One combined chart** — three capital lines (Chart.js).
3. **Three "critical dates"** — the year in which `passive ≥ target` (one per scenario).
4. **A three-sentence summary** — generated from the numbers.

## 2. Stack and language

- **Implementation:** a single static `index.html` with inline `<style>` and `<script>`. No bundlers, no npm.
- **Chart:** Chart.js v4 via CDN. The version and API are confirmed through `context7` MCP (`mcp__plugin_context7_context7__resolve-library-id` → `query-docs` for `chartjs/Chart.js`).
- **Fonts:** IBM Plex Mono for every number; IBM Plex Sans (or another clean humanist sans/serif) for body text — finalised in phase 6.
- **Document and UI language:** English. Parameter names and table column headers are also English.
- **How to run:** double-click `index.html` (no `fetch` of local files — there is no CSV).
- **Execution mode:** one phase at a time, with a review stop between phases.

## 3. Inputs (form)

Form fields, top to bottom:

| Field | Type | Unit | Validation | Default (for easy debugging) |
|---|---|---|---|---|
| `currentCapital` | number ≥ 0 | currency | required | `10000` |
| `monthlyIncome` | number ≥ 0 | currency/mo | required | `3000` |
| `monthlyExpenses` | number ≥ 0 | currency/mo | required, ≤ `monthlyIncome` *(soft warn, not error)* | `2000` |
| `horizonYears` | integer 1..60 | years | required | `30` |
| `targetPassiveIncome` | number > 0 | currency/mo *(in today's money)* | required | `4000` |

The scenario parameters (§4) **are not editable in the form** — they are baked into the code as a `SCENARIOS` constant. This is intentional: the task is about three fixed views of the future.

**Currency** is neutral (the input is just a number; the UI shows no symbol or just an indicative `$`). Multi-currency and FX are out of scope for this spec.

## 4. Three scenarios

```js
const SCENARIOS = {
  conservative: { return: 0.05, inflation: 0.06, incomeGrowth: 0.00 },
  realistic:    { return: 0.08, inflation: 0.05, incomeGrowth: 0.03 },
  aggressive:   { return: 0.12, inflation: 0.04, incomeGrowth: 0.07 },
};
```

`return` is the annual nominal return on capital. `inflation` is the annual growth of expenses and target. `incomeGrowth` is the annual growth of monthly income.

## 5. Forecast model

All amounts are nominal (no deflation applied). Discounting happens in exactly one place: the target is re-stated into the nominal terms of each year via the scenario's inflation (§5.3).

### 5.1. Year-`n` state (n = 0 — present moment)

- `capital[0]   = currentCapital`
- `income[0]    = monthlyIncome`
- `expenses[0]  = monthlyExpenses`
- `target[0]    = targetPassiveIncome`

### 5.2. Recurrence (per scenario)

```
savings[n]    = max(income[n] − expenses[n], 0) × 12
capital[n+1]  = capital[n] × (1 + return) + savings[n]
income[n+1]   = income[n]   × (1 + incomeGrowth)
expenses[n+1] = expenses[n] × (1 + inflation)
target[n+1]   = target[n]   × (1 + inflation)
```

**Timing convention:** contributions land **at end of year**. Capital first grows by `return`, then the year's savings are added. This is the simplest convention; alternatives (annuity-due, mid-year) are out of scope.

**`max(..., 0)`** — if expenses exceed income, savings are 0; "negative savings" do not draw down capital (consumption is outside the model).

### 5.3. Passive income and the target comparison

In year `n`:

```
passiveMonthly[n] = capital[n] × return / 12
```

The critical year for a scenario is **the smallest `n ∈ [0, H]` such that `passiveMonthly[n] ≥ target[n]`**. If no such `n` exists on the horizon, the critical year is `null` ("beyond horizon").

"Year" is an integer (see §5.4). Month-level granularity is out of scope.

### 5.4. Granularity

- The "capital by year" table has `H + 1` rows (`n = 0, 1, …, H`).
- The critical year is an integer number of years from the current year.
- The calendar year is shown as `currentYear + n`, where `currentYear` is taken from `new Date().getFullYear()` once at render time.

## 6. Outputs

### 6.1. Three tables

One per scenario. Columns:

| Column | Description |
|---|---|
| `year` | `n` |
| `calendarYear` | `currentYear + n` |
| `capital` | `capital[n]`, rounded to integer |
| `monthlyIncome` | `income[n]` |
| `monthlyExpenses` | `expenses[n]` |
| `passiveMonthly` | `capital[n] × return / 12` |
| `targetInflated` | `target[n]` |
| `coversTarget` | `Y` / empty |

The row where `coversTarget == Y` is visually highlighted (thin accent rule on the left). That's the "critical date" row.

### 6.2. Combined chart

One `<canvas>`, Chart.js, type `line`:

- **X** — year (`n = 0…H`), labelled by `calendarYear`.
- **Y** — capital, nominal.
- **Three lines**: `conservative`, `realistic`, `aggressive`. Colours are three distinguishable shades of one cold palette (final values in phase 6); legend on top or right.
- No area fill, thin lines, point markers only on the critical points.

### 6.3. Three critical dates

One block above the chart, three lines of the form:

```
Conservative — 2042 (in 16 years)
Realistic    — 2036 (in 10 years)
Aggressive   — 2031 (in 5 years)
```

If a scenario doesn't reach the goal on the horizon: `— beyond horizon (>H years)`.

### 6.4. Summary (3 sentences)

Auto-generated text below the dates. Template:

```
In the conservative scenario the goal is reached in {{cN}} years — this is the baseline.
Realistic — in {{rN}} years, {{cN−rN}} years sooner; this is the scenario to keep in mind as the default.
Aggressive — in {{aN}} years, conditional on a consistently high return of {{aReturn}} %, which is not historically guaranteed.
```

If any scenario lands "beyond horizon," that sentence is re-phrased, but the total is **always exactly three sentences**.

## 7. Six phases

### Phase 1 — Page skeleton

**Artifact:** `index.html` with all result blocks as placeholders.
**Does:** lays out the form with the §3 fields, three empty `<div>`s for tables (one per scenario), an empty `<div>` for the chart, one for the critical-dates block, one for the summary. CSS is minimal (readable flow, nothing polished).
**Does NOT do:** any computation, chart, or validation.
**Delivers:** the page opens, the form is visible, placeholder blocks carry clear `here will be …` text.

### Phase 2 — Pure calc logic

**Artifact:** a `forecast(inputs, scenarioParams) → { years: [...], criticalYear }` function inside `<script>`, plus a console-only debug block.
**Does:** implements §5 as a pure function (no DOM). The page calls the function with test inputs and emits `console.table(...)` for each of the three scenarios. Alongside that, three or four known edge cases are exercised via `console.assert`:
- `expenses > income` → `savings = 0`, capital still grows on return alone.
- `currentCapital = 0`, `savings > 0` → capital is monotonically non-decreasing, critical year exists on a long enough horizon.
- `target = ∞` (e.g. `1e9`) → `criticalYear === null`.
- Conservative with tiny capital and a large target → `criticalYear === null` at H = 30.

**Does NOT do:** anything on the page beyond console output.
**Delivers:** DevTools shows the three `console.table` outputs and the assertions are silent (no `Assertion failed:` lines).

### Phase 3 — Wire calc to form

**Artifact:** a `submit` handler on the form (and/or `input` for live mode — implementer's choice).
**Does:** reads the form, validates, calls `forecast` for each of the three scenarios, renders **three tables** into the corresponding `<div>`s per §6.1. Highlights the row with `coversTarget == Y`.
**Does NOT do:** chart, critical-dates block, summary.
**Delivers:** filling the form and clicking submit yields three tables. Changing a field and re-submitting redraws them.

### Phase 4 — Chart

**Artifact:** Chart.js v4 via CDN + the combined chart per §6.2.
**Does:** loads the CDN, draws one `<canvas>` with three line datasets. Colours are temporary (final values in phase 6). Before starting — `context7 MCP` to confirm the Chart.js v4 API: `resolve-library-id` → `query-docs` (`topic: "line chart multiple datasets"`).
**Does NOT do:** critical dates, summary.
**Delivers:** a chart under the tables; three distinguishable lines; X-axis = years, Y-axis = capital.

### Phase 5 — Interpretation

**Artifact:** the "3 critical dates" block (§6.3) and the "summary" block (§6.4).
**Does:**
- Extracts `criticalYear` per scenario from the `forecast` result.
- Renders the dates block above the chart (or above the tables — finalised in phase 6).
- Generates three summary sentences per the §6.4 template, substituting actual numbers.
- Marks the critical points on the chart with a single point marker per scenario (when present).
**Does NOT do:** visual polish.
**Delivers:** above/below the chart — three date lines and a three-sentence summary. If all three scenarios are "beyond horizon," the summary still re-phrases correctly to exactly three sentences.

### Phase 6 — UX polish

**Artifact:** final typography, palette, and layout. **Run the `frontend-design` skill** for distinctive styling; **verify via Playwright MCP** (or Chrome MCP) — screenshots at multiple viewports and console-message check.
**Does:** picks the final palette (three distinguishable shades for the chart lines, an accent colour for the critical-row highlight), introduces IBM Plex Mono for numbers, right-aligns table numerics with `tabular-nums`, sets vertical rhythm. Runs through Playwright at 1280px, 1024px, 768px viewports, takes screenshots, checks `browser_console_messages` — no errors and no warnings.
**Does NOT do:** change the model or the schema.
**Delivers:** the page looks editorial (like `passive-income` / `portfolio-journal`), console is clean at all three viewports.

## 8. Acceptance criteria

| # | Criterion | How to verify |
|---|---|---|
| AC1 | `index.html` exists, opens by double-click, no server required | Open locally, form is visible |
| AC2 | Form has exactly the 5 fields from §3, all required, defaults match §3 | Read DOM / DevTools |
| AC3 | Submit renders **three** tables (one per scenario) | DOM: 3 elements of class `.scenario-table` |
| AC4 | Each table has `H + 1` data rows (plus header) | `tr.length == H + 2` |
| AC5 | All 8 columns from §6.1 are present in each table | Read `thead` |
| AC6 | Numbers in `capital`, `passiveMonthly`, `targetInflated` are integer-formatted, no `NaN`/`undefined` | Pull `textContent`, regex `/^-?\d{1,3}(,\d{3})*$/` |
| AC7 | `coversTarget == Y` appears on the first row where `passiveMonthly ≥ targetInflated`, and nowhere before or after (zero or one marker per scenario) | Cross-check vs. a `console` recomputation |
| AC8 | Chart has exactly 3 datasets, labels `['conservative','realistic','aggressive']` | `chart.data.datasets.length === 3` |
| AC9 | Chart X-axis labels match `calendarYear` from the tables | Read chart config |
| AC10 | Critical-dates block — exactly 3 lines, format `<scenario> — <calendarYear> (in <N> years)` or `<scenario> — beyond horizon (>H years)` | regex over `innerText` |
| AC11 | Summary — exactly 3 sentences (3 sentence terminators in one block) | `summary.split(/\.\s+/).length === 3` (counting the trailing period) |
| AC12 | At horizon = 30 and defaults §3, scenario `realistic` matches a hand recomputation (±1 year) | Note expected number in `verify.js` / manual |
| AC13 | Chrome console — no errors or warnings at three viewports (1280, 1024, 768) | Playwright/Chrome MCP: `browser_console_messages` |
| AC14 | Phase boundaries actually stopped for user review | Process: each phase = a separate message with the delta and an explicit "ready for review" |

## 9. End-to-end verification

After phase 6:

1. **Open `index.html`** by double-click.
2. **Defaults**: fill the form with §3 values, submit. Confirm:
   - three tables (AC3);
   - each has `30 + 1 = 31` data rows (AC4 at H = 30);
   - the chart shows three diverging lines;
   - critical dates: 3 lines;
   - summary: 3 sentences.
3. **Edge case 1 — `expenses > income`**: `monthlyIncome = 2000`, `monthlyExpenses = 3000`. Savings must be 0, capital still grows. Critical year — later than at defaults.
4. **Edge case 2 — unreachable target**: `targetPassiveIncome = 50000`, `currentCapital = 1000`. For conservative the critical year must be `null` → block prints "beyond horizon".
5. **Edge case 3 — H = 1**: horizon 1 year. Tables — 2 data rows, chart — 2 points per line, everything renders without errors.
6. **Playwright MCP**:
   - `browser_navigate http://localhost:8847/index.html` (a tiny static server, since file:// is blocked)
   - `browser_resize 1280×800`, `browser_take_screenshot`, `browser_console_messages` → []
   - `browser_resize 1024×768`, repeat
   - `browser_resize 768×1024`, repeat
7. **AC1–AC14** — walk the list, every item PASS.

## 10. Out of scope

- Multi-currency and FX conversion.
- Taxes and fees (return is gross, no withholding).
- The withdrawal phase ("what happens after FI") — the model stops at the moment the goal is covered.
- Real-money mode (the whole arithmetic stays in nominal terms; the real-vs-nominal lens is applied only to the target).
- Month-level granularity — years only.
- User-editable scenario parameters (they are baked into the code).
- Persisting form state across reloads (no localStorage etc.).
- PDF / CSV export.

## 11. Critical files

| Path | What |
|---|---|
| `fi-forecast/spec.md` | This document (first artifact created) |
| `fi-forecast/index.html` | The only executable artifact; everything inline: HTML + `<style>` + `<script>` |

No subdirectories, no external `app.js`/`styles.css` — while the file fits, it stays in one piece.
