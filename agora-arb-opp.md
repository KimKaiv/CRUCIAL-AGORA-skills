# /agora-arb-opp — Cross-Market Arbitrage Opportunity Scanner

Scan all open OPER: markets for matched pairs (e.g. a 1D market and one dimension of a 2D market, or one dimension of a 2D market and one dimension of another 2D market, sharing the same variable and evaluation period), then report cross-market price differences at four levels of detail.

---

## Step 0 — Collect threshold from user

**IMPORTANT: Do this FIRST, before calling any other tools (including ToolSearch or list_markets).**

Call AskUserQuestion immediately with the following question and options. Do not proceed to Step 1 until the user has answered.

- Question: "What is the minimum price difference you want to flag as an arbitrage opportunity?"
- Header: "Arb threshold"
- Options (single-select):
  1. `0.02` — flag bins where |p_1D − p_marginal| ≥ 0.02
  2. `0.05` — flag bins where |p_1D − p_marginal| ≥ 0.05
  3. `0.075` — flag bins where |p_1D − p_marginal| ≥ 0.075
  4. `0.10` — flag bins where |p_1D − p_marginal| ≥ 0.10
  - Allow "Other" for a custom value typed by the user.

Store the result as **THRESHOLD**. Only then proceed to Step 1.

---

## Step 1 — Fetch live markets

Call `list_markets`. From the results, keep only markets where:
- `outcome_id` is null (market is still open), AND
- The market name starts with `OPER:`

---

## Step 2 — Classify and match markets into pairs

### Classification rules (apply in order — first match wins)

| Market name contains | Classification | Dimension of interest |
|---|---|---|
| `RONI-IODMI` | 2D market | RONI = `values[0]` (first dim) |
| `CYCLONES-HURR-TYPH` | 2D market | CAH (Atlantic hurricanes) = `values[0]` (first dim) |
| `RONI` | 1D market (RONI) | `values[0]` |
| `CYCLONES-ATLANTIC-HURRICANES` | 1D market (CAH) | `values[0]` |

### Pair-matching rules

For each 2D market, find a matching 1D market as follows:

**RONI-IODMI ↔ RONI:**
- Extract the `YYYY-MMM` suffix from the 2D market name (e.g. `RONI-IODMI-001-2026-SON` → `2026-SON`)
- Look for an open RONI 1D market whose name ends with the same `YYYY-MMM`
  (e.g. `RONI-005-2026-SON` → suffix `2026-SON`)
- If found: this is a matched pair. Record both market IDs, names, shared variable (RONI), and period.

**CYCLONES-HURR-TYPH ↔ CYCLONES-ATLANTIC-HURRICANES:**
- Extract the `YYYY` year suffix from the 2D market name (e.g. `CYCLONES-HURR-TYPH-2026` → `2026`)
- Look for an open CAH 1D market whose name ends with the same `YYYY`
- If found: matched pair. Shared variable = Atlantic hurricanes, period = YYYY season.

If no matched pairs are found across all open OPER: markets, print:
> "No matched pairs found among currently open OPER: markets."
and stop.

---

## Step 3 — Fetch prices for all matched-pair markets

For each matched pair, call `get_market_prices` on **both** the 1D market and the 2D market. Run these calls in parallel where possible.

---

## Step 4 — Compute the 2D marginal for the matched dimension

For each 2D market's price list:

1. Each price entry has the structure `{outcome_id, price, values, label}` where `values` is a list of two elements — one per dimension — each being either `[lo, hi]` (a continuous bin) or a string (categorical).
2. The **first dimension** (`values[0]`) corresponds to the 1D market's variable (RONI or Atlantic hurricanes).
3. **Group** all price entries by `values[0]`. Entries with identical `values[0]` belong to the same 1D bin.
4. **Sum** the `price` field within each group. This gives the **marginal probability** for that 1D bin.

The resulting marginal should sum to approximately 1.0 across all bins. If the sum deviates from 1.0 by more than 0.01, note this as a data anomaly in the output.

---

## Step 5 — Align bins and compute per-bin differences

For each matched pair:

1. Take the 1D market's price list. Each entry has `values = [[lo, hi]]` (or open-ended: `[null, hi]` or `[lo, null]`).
2. For each 1D bin, look up the marginal probability computed in Step 4 using `values[0]` as the key (exact match on the `[lo, hi]` pair).
3. Compute: `diff = |p_1D − p_marginal|` for each bin.
4. If a bin appears in one market but not the other (should not happen for same-spec markets), skip it and record a warning.

---

## Step 6 — Output the 4-level report

Present the full report as follows. Use the exact level structure shown.

---

### Level 1 — Matched pairs found: N

A table listing all matched pairs, for instance as follows (exact market matches and difference pair(s) will differ): 

| # | 1D Market (ID) | 2D Market (ID) | Shared variable | Period |
|---|---|---|---|---|
| 1 | `RONI-005-2026-SON` (115) | `RONI-IODMI-001-2026-SON` (154) | RONI | SON 2026 |
| 2 | `RONI-006-2027-DJF` (116) | `RONI-IODMI-002-2027-DJF` (155) | RONI | DJF 2027 |

---

### Level 2 — Largest single bin difference per pair

For instance as follows (exact market matches and difference pair(s) will differ): 

| Pair | Max \|p_1D − p_marginal\| | Bin |
|---|---|---|
| 115 ↔ 154 | 0.087 | [0.2, 0.4] °C |
| 116 ↔ 155 | 0.031 | [−0.2, 0.0] °C |

---

### Level 3 — Bins exceeding threshold (≥ THRESHOLD) per pair

For instance as follows (exact market matches and threshold-exceeding counts will differ): 

| Pair | Bins above threshold | % of total bins |
|---|---|---|
| 115 ↔ 154 | 4 of 32 | 12.5% |
| 116 ↔ 155 | 0 of 32 | 0% |

---

### Level 4 — Full bin-level difference tables

For each matched pair, print a separate table with one row per bin. Include all bins regardless of threshold.

Columns: `Bin | p_1D | p_marginal | |Δ| | flag`

- The `Bin` column shows the interval label (e.g. `< −3.0`, `[−3.0, −2.8]`, `≥ 3.0`).
- The `flag` column: leave blank if `|Δ| < THRESHOLD`; print `← above threshold` if `|Δ| ≥ THRESHOLD`.
- **Bold the entire row** (using markdown `**`) for rows where `|Δ| ≥ THRESHOLD`.

Example row above threshold:
```
| **[0.2, 0.4]** | **0.143** | **0.056** | **0.087** | **← above threshold** |
```

Print the marginal sum as a footer: `Marginal sum check: Σ p_marginal = X.XXX` (expected ~1.000).

---

## Step 7 — Interpretation notes

After the Level 4 tables, append an **Interpretation notes** section to the saved markdown report (and display it inline). Write one bullet per matched pair.

Each bullet should:
1. **Name the pair** by period (e.g. "Pair 1 (SON 2026):").
2. **Characterise the divergence pattern** — where is the 1D probability higher than the 2D marginal, and where is it lower? Describe the shape (e.g. "the 1D market concentrates mass in the [0.8, 1.6] range", "the 2D marginal is flatter and spreads more weight into negative bins").
3. **Quantify the key divergences** — cite the largest |Δ| bin and the direction of the gap (1D over- or under-weights relative to the 2D marginal).
4. **Connect to domain context** where the variable allows it:
   - For RONI: map the bin range to ENSO state (e.g. values ≥ 0.5 → El Niño-like; ≤ −0.5 → La Niña-like; near zero → neutral).
   - For CAH (Atlantic hurricanes): note whether the divergence is in the low, average, or high end of the historical hurricane-count distribution.
5. **Suggest a trading implication** in one sentence — e.g. which market looks cheap relative to the other for a given bin, or whether the 1D and 2D markets are telling inconsistent stories.

Keep each bullet to 3–5 sentences. Do not speculate beyond what the price data shows.
