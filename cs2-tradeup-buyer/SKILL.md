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

### Overview

The skill executes this deterministic loop:
1. Parse user input → get parameters
2. Find ONE trade-up opportunity
3. Scout ALL inputs for that opportunity (with systematic alternatives)
4. If ANY input group fails → report specific failure + ask user if they want to try a different trade-up
5. If ALL input groups succeed → proceed to confirmation & purchase

**Key principle:** You are scouting ONE trade-up completely before moving to the next. Do not auto-switch trade-ups. Ask the user first.

---

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

⚠️ **CRITICAL:** Pagination is mandatory. Do NOT stop after page 1 if it returns 0 results. Different pages contain different float ranges and prices. Continue paging until listings are found or market is exhausted.

#### 2a. Pagination loop (mandatory)

For each input in the opportunity:

```
PAGINATION_LOOP:
  page = 1
  collected_listings = []
  consecutive_zero_pages = 0

  WHILE (need more listings):
    Call FetchFloatData:
      itemName: input.itemName (without wear condition)
      wearCondition: input.wearCondition
      maxFloat: input.maxFloat (hard ceiling—do not modify)
      maxPrice: input.maxPrice (hard ceiling—do not modify)
      pageSize: 100
      page: page

    page_listings = response.listings

    IF (page_listings.empty):
      consecutive_zero_pages += 1
      IF (consecutive_zero_pages == 1):
        → Continue to page+1 (first empty page is normal)
      ELSE IF (consecutive_zero_pages >= 2):
        → Market exhausted. Exit loop.
      ELSE:
        → Continue to page+1
    ELSE:
      consecutive_zero_pages = 0
      collected_listings += page_listings
      IF (collected_listings.length >= input.Count):
        → Have enough. Exit loop.

    page += 1
    IF (page > 15):
      → Safety limit: market exhausted.

  END WHILE

  Return: collected_listings
```

#### 2b. Pagination rules (DO NOT SKIP)

1. **Page 1 returns 0 results?** → Continue to page 2, 3, 4...
   - Different pages have different float ranges
   - Strict constraints may skip early pages

2. **Prices are high on page 1?** → Continue paging.
   - Lower-priced items may be on later pages

3. **Float constraints not met?** → Continue paging.
   - Float availability varies across pages

4. **STOP pagination ONLY when:**
   - ✅ Collected enough listings (≥ input.Count), OR
   - ✅ 2+ consecutive empty pages (market exhausted), OR
   - ✅ Reached page 15+ (safety limit)

5. **NEVER assume "no listings"** based on page 1 alone.

#### 2c. Listing fields

Each listing has: `listingId`, `itemName`, `float`, `price`, `subtotalCents`, `feeCents`

#### 2d. Shortfall handling (systematic alternatives)

After primary item pagination exhausted (collected < needed):

```
collected = X (< input.Count)
alternatives = input.AlternativeSkins (ordered list, same collection/rarity)

FOR EACH alternative IN alternatives:
  IF (collected >= input.Count):
    → Input group satisfied. BREAK.

  collected_from_alt = pagination(alternative.itemName, alt.maxFloat, alt.maxPrice)
  collected += collected_from_alt

  IF (collected >= input.Count):
    → Input group satisfied. Return collected.

END FOR

IF (collected < input.Count after all alternatives):
  → Input group FAILED.
    Return: { status: "FAILED", collected: X, needed: input.Count, attempted: [primary + all alternatives] }
```

**Important:** If any alternative fails to produce ANY listings (0 collected across all pages), still try the next alternative. Exhausting all alternatives is what matters.

#### 2e. Input group failure → Trade-up abort decision

If ANY input group returns FAILED status:

1. **Report the failure clearly:**
   ```
   ❌ Trade-Up #24986065 CANNOT BE EXECUTED

   Input Group: 3x MAC-10 | Acid Hex (Minimal Wear)
   - Required: 3 items
   - Found: 0 items
   - Constraint: float ≤0.0812, price ≤R2.48
   - Attempted:
     ✗ MAC-10 | Acid Hex (primary): 0 listings on pages 1-2
     ✗ Galil AR | Acid Dart (alt 1): 0 listings on page 1
     ✗ P90 | Mustard Gas (alt 2): 0 listings on pages 1-2
     ✗ [... all alternatives exhausted]

   Reason: Current market prices exceed opportunity constraints.
   ```

2. **Ask user explicitly:**
   ```
   Would you like to:
   A) Try a different trade-up (I'll pick the next best opportunity)
   B) Increase budget and retry (specify new maxCost)
   C) Abort
   ```

3. **Do NOT auto-switch.** Wait for user input before proceeding to a different trade-up.

---

### Step 2-final — Validate completeness

After Step 2 completes (all input groups have been scouted):

```
FOR EACH input_group IN opportunity.inputs:
  IF input_group.status == "FAILED":
    → Jump to Step 2e (failure handling)
    → Do not proceed to Step 3.

IF all input_groups.status == "COLLECTED":
  → Proceed to Step 3 (confirmation).
```

---

### Step 3 — Report findings and ask for confirmation

**You only reach this step if ALL input groups were successfully sourced** (validated in Step 2-final).

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
