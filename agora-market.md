Display detailed price information for market ID $ARGUMENTS.

> **Note:** 2D market rendering requires Python with numpy (available via `requirements.txt`).

**Steps:**
1. Call `list_markets` to get the market name and liquidity_factor for market $ARGUMENTS
2. Call `get_market_prices` with market_id=$ARGUMENTS
3. Look up the market category spec: match the market name against the regex patterns in `knowledge/specs/_index.json`, then read `knowledge/specs/<slug>.md` if a match is found. If no match, proceed without spec context.
4. Detect dimensionality: inspect the `values` field of the first outcome in the prices list.
   - If `len(values) == 1` → **1D market** → follow the **1D Display** section.
   - If `len(values) == 2` → **2D market** → follow the **2D Display** section.

---

## 1D Display

Render an ASCII barplot of the outcome price distribution:
- Each outcome occupies one row
- Bar width is proportional to price (scale so the largest bar is ~40 characters wide)
- Show the numeric price (as a percentage) and the outcome label next to each bar
- Highlight the modal outcome (highest price) with an arrow or marker
- Example row format:
  ```
  450 to 500  ████████████████████████████████████████  11.7% ◄ modal
  500 to 550  ████████████████████████████████████       11.3%
  Under  300  ████████████                                7.1%
  ```

**Annotation paragraph** (below the barplot):
- What the market is measuring and why it matters — draw on the spec's "What it measures" section if available
- What the current price distribution implies (e.g. implied median, most likely range) — use the outcome-space interpretation table from the spec where relevant (e.g. ENSO phase thresholds for RONI markets)
- Notable features: skew, fat tails, bimodality, concentration of probability
- For climate/weather markets: relate the distribution to known baselines or model forecasts where relevant
- Settlement: note the data source and key settlement date rule from the spec

---

## 2D Display

**Grid reconstruction**

Reconstruct the 2D probability grid from the flat outcome list:
- **Dim 1 (rows)** = unique sorted values of `values[0]` across all outcomes. Sort by lower bound; treat `null`/`None` lower bound as −∞ (puts "Under X" first) and `null`/`None` upper bound as +∞ (puts "Over X" last).
- **Dim 2 (cols)** = unique sorted values of `values[1]`, same sorting rule.
- Build `grid[r][c] = price` for the outcome whose dim-1 value = row r and dim-2 value = col c.

Row label format (use lower bound of each bin):
- `[null, hi]` → `<{hi}`
- `[lo, hi]`   → `{lo}`
- `[lo, null]` → `≥{lo}`

Column headers: same convention, abbreviated to ≤5 chars per header.

---

**ASCII heatmap**

Find the **mode** = the single cell with the highest probability in the grid.
Divide the interval `[0, mode]` into 5 equal-width bands and assign shading:

| Char | Band | Range           |
|------|------|-----------------|
| `·`  | 1    | [0, mode/5)     |
| `░`  | 2    | [mode/5, 2×mode/5) |
| `▒`  | 3    | [2×mode/5, 3×mode/5) |
| `▓`  | 4    | [3×mode/5, 4×mode/5) |
| `█`  | 5    | [4×mode/5, mode] |

Layout: row labels on the left; dim-2 column headers above the grid; one character per cell, no separator between cells in the same row. Example sketch (RONI-IODMI):

```
         IODMI (°C) →
         <-2 -1.8 -1.6 … +2.8 ≥+3
RONI   <-3.0  ··················
(°C)   -2.8   ·░··············
↓      -2.6   ··▒·············
       …
       ≥+3.0  ················

Legend: · very low  ░ low  ▒ mid  ▓ high  █ peak
```

---

**Marginal distributions**

Compute marginals from the grid:
- **Dim-1 marginal**: `p[r] = Σ_c grid[r][c]`
- **Dim-2 marginal**: `p[c] = Σ_r grid[r][c]`

Display each as a compact ASCII barplot (max bar ~25 chars wide, modal bin marked).
Use the actual index names from the spec where available, e.g.:

```
RONI marginal:
  <-3.0  ██                    0.8%
  -2.8   ████                  1.6%
  …
  +0.2   █████████████████████████  9.4% ◄ modal
  …

IODMI marginal:
  …
```

---

**Top-5 joint outcomes**

List the five cells with the highest probability:

```
Top joint outcomes:
  1. RONI [0.6, 0.8] × IODMI [0.2, 0.4]:  1.82%
  2. RONI [0.4, 0.6] × IODMI [0.0, 0.2]:  1.71%
  …
```

---

**2D Annotation paragraph** (below all displays):
- What the two dimensions measure and why their joint distribution matters — draw on the spec's "What it measures" section
- The peak joint cell (mode) and surrounding probability mass (e.g. fraction within ±1 bin of the mode in each dimension)
- Dim-1 and dim-2 marginal summaries: implied median and modal bin for each; ENSO phase / IOD phase interpretation (for RONI-IODMI) or historical base-rate comparison (for HURR-TYPH), using thresholds from the spec
- Whether the joint distribution shows evidence of correlation (e.g. the mode is off-diagonal relative to the independent product of marginals)
- Settlement: note both data sources and the key settlement date rule from the spec

---

**Metadata line:**
- Show: market_id, liquidity_factor, last price update timestamp, and settlement status

**If asked to save or package the output**, write it to a file in the project root using this naming pattern:

```
mkt{market_id}_{type_label}_{YYYY-MM-DD}.md
```

Derive `type_label` from the market name:

| Market name contains | type_label |
|---|---|
| `RONI-IODMI` | `RONIxIODMI` |
| `CYCLONES-HURR-TYPH` | `HURRxTYPH` |
| `RONI` (not RONI-IODMI) | `RONI` |
| `AISMR` | `AISMR` |
| `CYCLONES-ATLANTIC-HURRICANES` | `CAH` |

Example: market 154 (RONI-IODMI), saved on 2026-05-31 → `mkt154_RONIxIODMI_2026-05-31.md`

**End your response with:**
> Use `/agora-trade $ARGUMENTS` to trade on this market.
