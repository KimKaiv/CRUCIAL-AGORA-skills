Fetch all visible AGORA prediction markets using the `list_markets` MCP tool, then present them attractively.

**Display format:**
- Group markets by organization (organization_id), with a heading for each group
- For each market show: market_id, market name, liquidity_factor, and settlement status
- Mark settled markets (outcome_id is non-null) with [SETTLED] and open markets with [OPEN]

**Annotate market names using domain knowledge:**
- Decode acronyms and seasonal codes:
  - RONI = Regional Ocean–atmosphere Index (precipitation/climate index)
  - AISMR = All-India Summer Monsoon Rainfall
  - CYCLONES-ATLANTIC-HURRICANES = Atlantic named storm / hurricane count
  - CYCLONES-HURR-TYPH = Combined hurricane + typhoon count
  - DJF = December–January–February season
  - MAM = March–April–May season
  - JJA = June–July–August season
  - SON = September–October–November season
  - YYYY = year, e.g. 2025 or 2026
  - OPER: = Operational market (real forecasting use)
  - DEMO: = Demonstration market
- For each market, add a one-sentence plain-English description of what is being forecast
- Where a spec file exists in `knowledge/specs/`, use the first sentence of "What it measures" as the description; otherwise derive from the market name alone

**End your response with:**
> Use `/agora-market <market_id>` to see the full price distribution for a specific market.
