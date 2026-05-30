Display detailed price information for market ID $ARGUMENTS.

**Steps:**
1. Call `list_markets` to get the market name and liquidity_factor for market $ARGUMENTS
2. Call `get_market_prices` with market_id=$ARGUMENTS
3. Look up the market category spec: match the market name against the regex patterns in `knowledge/specs/_index.json`, then read `knowledge/specs/<slug>.md` if a match is found. If no match, proceed without spec context.

**Display:**

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

**Metadata line:**
- Show: market_id, liquidity_factor, last price update timestamp, and settlement status

**End your response with:**
> Use `/agora-trade $ARGUMENTS` to trade on this market.
