---
name: cs2-tradeup-buyer
description: >
  Portable, MCP-first workflow to find and execute CS2 trade-up input purchases end-to-end.
  Use this skill whenever the user wants profitable trade-up scouting and optional auto-purchase
  with budget/float constraints.
  Triggers on: /cs2-tradeup-buyer <maxCost>, "find and buy tradeup inputs", "scout tradeup under R<X>",
  "buy tradeup inputs for me", or any request to automate trade-up purchase flow.
---

# CS2 Trade-Up Buyer

Automates the full workflow: find a target trade-up opportunity within budget/ranking constraints -> scout Steam market listings
that meet float and price constraints -> substitute with same-collection alternatives when needed -> confirm
with user -> execute purchases.

## Portability Guarantees

- Do not rely on local repository files, local project paths, or local source code.
- Operate only through the connected MCP server and its exposed tools/resources.
- Do not require users to provide credentials in chat; assume server-side auth/credential store.
- If details are missing from this document, discover them from MCP tool metadata and schemas at runtime.

## Invocation

```
/cs2-tradeup-buyer <maxCost>
```

`maxCost` is the maximum total cost in ZAR (e.g. `10` = R10). If omitted, ask the user for it.

Optional intent examples:
- "second most profitable under R10 with unique output"
- "top 3 unique outputs under R25"

## Authentication

This skill uses the MCP server's saved credentials (server-side credential store). The skill must not prompt users to provide Steam/API credentials or require credential files to be passed in workflows — assume server-side credentials are available and used for all execute calls.

## Connection Strategy

This skill **must always use** the MCP server at: `https://cs2marketbot.lloydindevelopment.work/mcp`

1. Ensure the MCP server is configured and connected in the client session.
2. If the server is not configured, request that it be set up with the URL above before proceeding.
3. Discover capabilities first via MCP tool listing and tool schemas.
4. Map required operations by capability, not by hard-coded names.

Required capabilities to locate dynamically:
- Opportunity search/listing (filterable by cost, sorting, currency, pagination)
- Float/listing fetch for specific item + wear with pagination
- Purchase execution from explicit listing payloads
- (Optional) collection/alternative lookup, if alternatives are not embedded in opportunity inputs

Common tool names may include:
- `get_trade_up_opportunities`
- `fetch_float_data` or `FetchFloatData`
- `execute_trade_up_input_purchases`

Never fail just because names differ; use tool descriptions + schemas to pick the equivalent.

---

## Workflow

### Step 0 — Parse user target selector

Extract these request controls from the prompt:
- `maxCost` (required)
- `currency` (default from server/tool; if supported, prefer `ZAR` when the user asks in R)
- `rank` (default `1`; for "second most" use `2`)
- `uniqueByOutput` (default `true`; set to false only if user explicitly wants "all variations" or "including duplicates")
- any extra filters (trade-up type, min profit, etc.)

### Step 1 — Find ranked opportunities

Call the discovered opportunity tool with:
- `maxCost` = user budget
- `sortBy` = `"profit"`
- `currency` = user requested currency (or omit if tool/server does not support currency parameter)
- `pageSize` = 5 (default; increase if user asks for "top N" or "all options")
- `uniqueByOutput` = true (default; set false only if user wants all variations including same outputs)

Selection logic:
- Sort descending by profitability (or rely on server sort if guaranteed).
- Pick the 1st (or Nth) unique output skin by profitability.
- Output identity key preference:
  1) stable output skin ID if available
  2) output market hash name
  3) output name + wear condition

For "second most profitable under R10 with unique output":
- set `maxCost=10`, `rank=2`, `uniqueOutput=true`, `currency=ZAR` (if supported).

Display selected opportunity:
- Trade-up ID, type (Single / Multi), profit %, total cost, best output name
- For each input group: item name, wear condition, collection ID, rarity ID, required float, max price per item, quantity needed

> **Single** type: always 10× of one input type.
> **Multi** type: multiple distinct input types each with their own quantity (e.g. 8× item A + 2× item B).

If no opportunities found, tell the user and stop.

---

### Step 2 — Scout Steam market for each input

For each input in the opportunity, call FetchFloatData with:
- `itemName` = input item name (without wear condition)
- `wearCondition` = input wear condition
- `pageSize` = 100 (or adapt based on availability)
- Inline constraints (passed to FetchFloatData):
  - `maxFloat` = input.maxFloat (the hard ceiling from the opportunity; do not modify)
  - `maxPrice` = input.maxPrice (the per-item price ceiling from the opportunity; do not modify or calculate)

FetchFloatData returns only listings that pass both constraints (float ≤ maxFloat AND price ≤ maxPrice).

**Pagination strategy:** Automatically paginate through all results (incrementing `page` parameter). Continue pagination indefinitely across multiple pages:

**Stop pagination ONLY when:**
1. You've collected enough listings to meet the required quantity, OR
2. A page returns zero listings AND the previous page already showed increasing prices (market exhausted)

**Never stop pagination when:**
- A page returns 0 listings but prices haven't been checked yet—continue to next page
- Float constraints aren't met on current page—continue to next page (float ranges vary across pages)
- Prices seem high—continue paging (different float ranges mean different prices available further down)

**Parallel page fetching (optional optimization):** Pages are independent and safe to fetch in parallel using subagents. The web proxy handles Steam rate limits internally, so fetching multiple pages concurrently (e.g., pages 1-5 or 1-10 at once) will not trigger rate limits and significantly speeds up scouting. Use this approach when sourcing multiple inputs—pages can be gathered in parallel and merged by float/price.

Each listing has: `listingId`, `itemName`, `float`, `price`, `subtotalCents`, `feeCents`.

**Shortfall handling** (if collected listings < input.Count after pagination exhausted):
- Check `input.AlternativeSkins` — these are same-collection/same-rarity substitutes with the same adjusted float
- Each alternative has `itemName`, `wearCondition`, `maxItemFloat` (max float to match adjusted float), and `lowestPrice`
- Call FetchFloatData for each alternative and apply the same pagination strategy
- Alternative listings must have float ≤ alternative.maxItemFloat

If any input group yields **0 listings found** across all pages and alternatives are insufficient/unavailable, warn the user — the trade-up cannot be executed without all inputs.

---

### Step 3 — Report findings and ask for confirmation

Show a clear pre-purchase summary:

```
=== Trade-Up: <output name> ===
Type: Multi | Profit: 125% | Total Cost: <currency><amount>

INPUT GROUP 1 — need 8, found 7 ✅ / 1 ⚠️ short
  ✅ MP9 | Slide (MW)         Listing 630079... | Float: 0.0811 | <currency>0.61
  ✅ MP9 | Slide (MW)         Listing 631205... | Float: 0.0799 | <currency>0.58
  ✅ AUG | Storm (MW)         Listing 987654... | Float: 0.0804 | <currency>0.63   ← substitute
  ... (all collected)
  ⚠️ Only 7/8 found — 1 input missing

INPUT GROUP 2 — need 2, found 2 ✅
  ✅ AUG | Snake Pit (MW)     Listing 654321... | Float: 0.0765 | <currency>2.10
  ✅ AUG | Snake Pit (MW)     Listing 789012... | Float: 0.0770 | <currency>1.95

Total to spend: <currency><sum>

Proceed with purchase? (yes/no)
```

If any input group has **0 listings found**, warn prominently — the trade-up cannot be executed. Let the user decide to proceed with partial inputs or abort.

---

### Step 4 — Execute purchases

Only after the user confirms, call the discovered purchase execution tool with the **exact listings** collected during scouting — not the opportunity object. Pass each listing fields exactly as required by the tool schema (typically `listingId`, `itemName`, `price`, `float`, and if present `subtotalCents` + `feeCents`).

```json
[
  { "listingId": "646967639220103523", "itemName": "AUG | Sweeper (Minimal Wear)", "price": 0.49, "float": 0.0848, "subtotalCents": 42, "feeCents": 7 },
  { "listingId": "623323741174731345", "itemName": "AUG | Sweeper (Minimal Wear)", "price": 0.49, "float": 0.0700, "subtotalCents": 42, "feeCents": 7 },
  ...
]
```

If the float/listing tool returns extra pricing fields (for example `subtotalCents`, `feeCents`, fees, taxes, or converted totals), pass them through exactly when required by the purchase tool schema.

Report: items purchased, total spent, duration, any errors.

---

## Important constraints

- **Filtering happens at FetchFloatData** — call FetchFloatData with `maxFloat` and `maxPrice` parameters to get pre-filtered listings from Steam. The service applies constraints at fetch time (inline filtering).
- **Float constraint** — pass the exact `maxFloat` from the opportunity (input.maxFloat) to FetchFloatData. This is a hard ceiling, not a relaxed tolerance. The opportunity specifies the maximum float to preserve the trade-up's profitability and hit probability.
- **Price constraint** — pass the exact `maxPrice` from the opportunity (input.maxPrice) to FetchFloatData. This is a hard ceiling pre-calculated by the opportunity to preserve profitability. Do not recalculate or adjust it.
- **Pagination is mandatory and requires persistence** — You MUST page through multiple pages. The first page returning 0 results does NOT mean there are no listings. Continue paging aggressively:
  - Page 1 returns 0 results? → Page 2, Page 3, Page 4, ... continue
  - Float constraints not met on page 1? → Continue paging (float ranges vary across pages)
  - Prices too high on page 1? → Continue paging (earlier pages may have stricter float constraints; later pages show different float ranges with different prices)
  - **Only stop when:** You've collected enough listings OR you've confirmed the market is exhausted (e.g., paged 10+ pages with zero results AND prices keep increasing)
- **Currency behavior** — all costs are in ZAR; the server stores and returns prices in ZAR.
- **Rate limit handling** — the server's FloatScraperService handles Steam rate limits internally (throttle delays, retry logic). If rate limited, FetchFloatData returns what it could fetch in the current request.

## Missing Info Policy

If any required field/behavior is unclear:
1. Inspect MCP tool schema and descriptions.
2. Run a minimal probe call with safe limits (`pageSize=1`) to infer response structure.
3. Continue the workflow using discovered fields.
4. Only ask the user when MCP discovery cannot resolve ambiguity.
