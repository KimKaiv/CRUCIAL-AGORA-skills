Refresh the market category specification files in `knowledge/specs/` by fetching the latest content from crucialab.net. Also discovers any new OPER market categories not yet covered by `_index.json` and proposes additions for confirmation.

---

## Step 0 — Discovery

### 0a — Bootstrap if needed

- If `knowledge/specs/` does not exist, create it.
- If `knowledge/specs/_index.json` does not exist, start with an empty patterns list `{"patterns": []}` in memory only — do not write yet.
- Otherwise read the existing `_index.json`.

### 0b — Extract live OPER categories

Call `list_markets`. Keep only markets whose name starts with `OPER:`. For each, extract the **category root** by:

1. Stripping the `OPER: ` prefix.
2. Stripping any trailing series/year/season suffix. The suffix begins at the first component (reading right-to-left, splitting on `-`) that is a 4-digit year (e.g. `2026`), a 3-digit series number (e.g. `001`), or a seasonal code (`SON`, `DJF`, `MAM`, `JJA`, `JJAS`). Discard that component and everything after it.

Examples:
- `RONI-IODMI-001-2026-SON` → `RONI-IODMI`
- `RONI-005-2026-SON`       → `RONI`
- `CYCLONES-HURR-TYPH-2026` → `CYCLONES-HURR-TYPH`
- `CYCLONES-ATLANTIC-HURRICANES-2025` → `CYCLONES-ATLANTIC-HURRICANES`
- `AISMR-2026-JJAS`         → `AISMR`

Deduplicate to get the set of unique live category roots.

### 0c — Identify gaps

For each live category root, test whether any existing `_index.json` entry's `regex` matches it (case-insensitive substring match — same logic as `_resolve_spec_slug` in `agora_mcp/server.py`). Collect **unmatched categories** — those with no covering entry.

If there are **no unmatched categories**, print:
> Discovery: all live OPER categories are covered by `_index.json`. Proceeding to refresh.

Then skip to Step 1.

### 0d — Scrape crucialab.net/market/

Fetch `https://www.crucialab.net/market/` using WebFetch with this prompt:
> "List all market spec pages linked from this page. For each, extract: the link URL (in the form /market/\<slug\>/), the page title or heading, and any short description. Return as a structured list."

### 0e — Match unmatched categories to URLs

For each unmatched category:
- Compare its name against the titles and descriptions of the crucialab.net pages from step 0d.
- If a confident match is found: record `source_url = https://www.crucialab.net/market/<slug>/`.
- If no confident match: record `source_url = null` and flag as **⚠ URL unresolved**.

### 0f — Build proposed entries and determine ordering

For each unmatched category, construct:
```json
{
  "regex": "<CATEGORY-ROOT>",
  "slug": "<category-root-lowercased-with-hyphens>",
  "source_url": "<resolved URL or null>"
}
```

Merge the new entries with the existing patterns list. Sort the combined list by regex length **descending** (longer = more specific first), then alphabetically within the same length. This preserves the rule that e.g. `RONI-IODMI` must precede `RONI`.

### 0g — Show proposed changes and confirm

Present a diff table:

```
New entries proposed for _index.json:

  regex                        slug                  source_url
  ──────────────────────────── ───────────────────── ──────────────────────────────────
  NEWINDEX                     newindex              https://www.crucialab.net/market/newindex/
  ANOTHER-CAT                  another-cat           ⚠ URL not resolved — please provide manually

Full updated _index.json ordering after insertion:
  1. RONI-IODMI                  (existing)
  2. CYCLONES-HURR-TYPH          (existing)
  3. CYCLONES-ATLANTIC-HURRICANES (existing)
  4. NEWINDEX                    ← new
  5. RONI                        (existing)
  6. AISMR                       (existing)
  7. ANOTHER-CAT                 ← new (URL pending)
```

Ask:
> "Shall I update `_index.json` with these additions? You can also supply any missing URL marked ⚠ before confirming. Reply 'yes' to proceed, 'no' to skip the discovery update and continue refreshing existing specs only, or provide corrected URLs."

**Only write `_index.json` after explicit confirmation.** If the user provides corrected URLs, substitute them before writing.

### 0h — Write updated `_index.json`

On confirmation, write the updated `_index.json`. Then continue to Step 1, which now covers the newly added slugs as well.

---

## Step 1 — Read index

Read `knowledge/specs/_index.json` to get the list of slugs and their source URLs.

## Step 2 — Fetch each spec

For each entry in `_index.json`, fetch the `source_url` using WebFetch with this prompt:
> "Extract all structured content: market name/pattern, what it measures, outcome space (type, bins, range, bin width), data sources (name + URL), settlement rules, and any trading notes. Return as structured markdown sections."

## Step 3 — Write spec files

For each successfully fetched slug, rewrite `knowledge/specs/<slug>.md` using this template:
```
# <Full Market Name>

**AGORA market name pattern:** `<pattern from CLAUDE.md taxonomy>`

**Source URL:** <url>
**Last fetched:** <today's date YYYY-MM-DD>

---

## What it measures
<description>

## Outcome space
<type, bins, range, bin width, open-ended terminals>

## Data sources
<primary and secondary sources with URLs>

## Settlement rules
<when and how the market resolves>

## Trading notes
<platform constraints, useful external forecasts, caveats>
```

## Step 4 — Handle failures

If a URL returns no usable content (page not yet published, 404, etc.), update the file's `Status:` line to reflect that and preserve the existing placeholder content — do not blank the file.

## Step 5 — Check historical

Also check `knowledge/historical/` — if any historical market URLs in `_index.json` exist (future extension), fetch and update those too.

## Step 6 — Report summary

Report a summary table:

| Slug | URL | Outcome |
|------|-----|---------|
| cah  | ... | Updated / No change / Failed |
| roni | ... | Updated / No change / Failed |
| aismr| ... | Updated / Placeholder preserved (not yet published) |

**Do not** fetch specs on every skill invocation — this command is the only intended refresh trigger.
