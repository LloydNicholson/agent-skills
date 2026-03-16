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

## Error Handling

**Note:** The MCP server is now stateless, so session timeouts (-32001) will not occur. Tool failures will be rare, but if they do happen:

- **Network/transient errors**: Apply exponential backoff retry (1s, 2s, 4s) and report to user with all collected data.
- **Rate limiting**: Pause for the indicated duration and retry.
- **Invalid parameters**: Log the exact error and ask user to clarify or adjust constraints (e.g., float range, budget).

All operations are idempotent:
- Re-running `fetch_float_data` on the same page/item is safe and produces consistent results.
- Re-running `execute_trade_up_input_purchases` with the same listing IDs is safe (no duplicate charges).

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

**`lastHours` filter — schedule-aware default:**
Trade-up opportunities are recalculated **once daily at ~06:00 SAST (UTC+2)**. When choosing `lastHours`:
- Do **not** use `lastHours` by default — omitting it returns all available opportunities regardless of age.
- If the user invokes this skill **within ~20 hours of 06:00 SAST** (i.e. same calendar day), the freshest opps are from that morning's run. Omitting `lastHours` is fine.
- Only use `lastHours` if the user explicitly asks for "fresh" or "recent" opportunities, or if you suspect the opportunity DB may contain stale data from a previous day. In that case use `lastHours=20` to cover the full window since the last 06:00 SAST run.
- **Never use `lastHours=1` or `lastHours=6`** — these will almost always return no results since the batch runs once a day, not continuously.

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

### Step 2 — Scout Steam market for each input (PARALLEL PAGINATION)

⚠️ **CRITICAL:** Pagination is mandatory. Do NOT stop after page 1 if it returns 0 results. Different pages contain different float ranges and prices. Continue paging until listings are found or market is exhausted.

⚠️ **FLOATS CAN NEVER STOP THE LOOP:** The only conditions that terminate pagination are: (1) the collected listings reach the required count for the input, or (2) every listing on the active page violates the per-item price ceiling. Float filtering is only a gatekeeper—not a stopping condition. Keep paging indefinitely until you encounter one of the two terminal states above.

#### 2a. Parallel input scouting (key optimization)

**For Multi-type trade-ups with multiple input groups:** Instead of scouting inputs sequentially (Item A pages 1→N, then Item B pages 1→N), **spawn one subagent per input group**. Each subagent handles pagination for its assigned item in parallel.

**Example:** Trade-up with 2 inputs:
- ✅ Spawn Task 1: Scout Item A (pages loop until goal or price ceiling hit)
- ✅ Spawn Task 2: Scout Item B (pages loop until goal or price ceiling hit)
- 🔄 Both tasks run simultaneously
- ⏳ Main skill waits for all tasks to complete
- 🔗 Merge results from all tasks

For **Single-type trade-ups** (10× same item), apply pagination loop once (no parallelization needed).

#### 2b. Pagination loop (mandatory with explicit state tracking)

⚠️ **CRITICAL: This skill has NO page limit.** You will page indefinitely (100+, 500+ pages if needed) until one of two conditions:
1. **Collected goal:** You have enough items → STOP
2. **Price ceiling hit:** ALL items on a page exceed maxPrice → STOP

There is NO OTHER stopping condition. Not "page 15 safety limit", not "consecutive empties", not "arbitrary depth". Only goal or price ceiling.

**BEFORE starting pagination for ANY item, initialize this state table:**

```
ITEM: [itemName]
CONSTRAINT: float ≤ [maxFloat], price ≤ [maxPrice]
NEEDED: [input.Count] items

PAGINATION STATE:
| Page | Listings Found | Consecutive Empty | Reason for Stop? | ACTION |
|------|----------------|-------------------|------------------|--------|
|      |                |                   |                  |        |
```

**For Multi-type trade-ups, spawn subagents (one per input group):**

```
# Spawn all inputs in parallel (all at once, not sequentially)

FOR EACH input_group in opportunity.inputs:
  SPAWN SUBAGENT TASK:
    Agent type: task (deterministic work, can verify idempotency)
    Prompt: "Scout the Steam market for [input.itemName] (wear: [input.wearCondition])
             following this pagination workflow:
             
             - Item: [input.itemName]
             - Wear: [input.wearCondition]
             - Need: [input.Count] items
             - Constraints: float ≤ [input.maxFloat], price ≤ [input.maxPrice]
             - Output market price: R[opportunity.bestOutput.Price]
             - Max cost per item: R[input.maxPrice]
             
             Pagination rules:
             1. Start page 1, fetch pages sequentially
             2. STOP when: (a) collected [input.Count] items, OR (b) all items on a page exceed price ceiling
             3. NO arbitrary page limits — keep paging until one stop condition is met
             4. Use cs2-market-bot-fetch_float_data with all parameters specified below
             
             Call fetch_float_data with:
             - itemName: [input.itemName]
             - wearCondition: [input.wearCondition]
             - maxFloat: [input.maxFloat]
             - maxPrice: [input.maxPrice]
             - expectedOutputValue: [opportunity.bestOutput.Price]
             - pageSize: 100
             - page: <starting from 1, increment each call>
             
             Return a JSON with:
             {
               'item_name': '[input.itemName]',
               'status': 'success' | 'shortfall',
               'collected_count': <number>,
               'listings': [<listing objects>],
               'total_pages': <pages accessed>,
               'stop_reason': 'goal_reached' | 'price_ceiling_hit',
               'final_cost': <sum of prices>
             }
    Mode: background (so all subagents run in parallel)

# Wait for all subagents to complete
WAIT FOR ALL TASKS

# Merge results when all complete
```

**Execute the deterministic pagination loop (for inline or subagent execution):**

```
PAGINATION_LOOP:
  page = 1
  collected_listings = []
  price_ceiling_hit = false
  checkpoint = null

  LOOP:
    ✓ BEFORE EACH CALL: Log current state (page #, total collected so far)

    💾 CHECKPOINT SAVE: Before calling FetchFloatData, save checkpoint state
    checkpoint = {
      "operation": "fetch_float_data",
      "item_name": input.itemName,
      "wear_condition": input.wearCondition,
      "page": page,
      "items_collected": collected_listings.length,
      "max_float": input.maxFloat,
      "max_price": input.maxPrice,
      "timestamp": current_time
    }

    Call FetchFloatData:
      itemName: input.itemName (without wear condition)
      wearCondition: input.wearCondition
      maxFloat: input.maxFloat (hard ceiling—do not modify)
      maxPrice: input.maxPrice (hard ceiling—do not modify)
      expectedOutputValue: opportunity.bestOutput.Price (current market price of the trade-up output skin)
      pageSize: 100
      page: page

    ❌ IF ERROR (any kind):
      ✓ Log the error details
      ✓ Apply exponential backoff: wait 1s, then 2s, then 4s (3 retries max)
      ✓ After 3 retries fail: report to user with all items collected so far
      ✓ Checkpoint state allows resuming from same page if user wants to retry manually

    page_listings = response.listings
    page_count = length(page_listings)

    ✓ AFTER EACH CALL: Update state table with [page number], [listings found]
    ✓ CHECKPOINT UPDATE: If page succeeded, update checkpoint with new page number and items collected

    DECISION TREE:

    IF (page_count > 0):
      ✓ Log: "✅ Page [page]: [page_count] listings found (within float/price constraints)"
      collected_listings += page_listings

      IF (collected_listings.length >= input.Count):
        ✓ Log: "🎯 COLLECTED GOAL: Have [collected_listings.length] items (need [input.Count])"
        BREAK  ← Exit pagination loop, proceed to next input
      ELSE:
        ✓ Log: "📊 Progress: [collected_listings.length]/[input.Count] collected. Continuing..."
        page += 1

    ELSE IF (page_count == 0):
      ✓ Log: "📄 Page [page]: 0 listings found on this page"
      ✓ CHECK: Did FetchFloatData indicate ALL listings on this page exceed price ceiling?

      IF (response indicates all items exceed price ceiling):
        ✓ Log: "💰 PRICE CEILING HIT: All listings on page [page] exceed R[maxPrice]. Market cannot go lower."
        price_ceiling_hit = true
        BREAK  ← Exit pagination loop
      ELSE:
        ✓ Log: "ℹ️ No listings pass float constraints on this page, but price is OK. Float distribution varies. Continuing..."
        page += 1

  END LOOP

  ✓ FINAL LOG:
    ```
    === PAGINATION COMPLETE ===
    Item: [itemName]
    Collected: [collected_listings.length]/[input.Count]
    Pages accessed: 1-[page-1]
    Stop reason: [GOAL REACHED / PRICE CEILING HIT / STILL SEARCHING]
    Status: [ENOUGH / SHORTFALL]
    Last checkpoint: page [checkpoint.page], [checkpoint.items_collected] items collected
    ```

Return: collected_listings
```

#### 2c. Input independence & alternative sequencing

- Each input group maintains its own pagination loop and state. If one group hits the price ceiling (stop_reason=`price_ceiling_hit`), mark it as blocked but **do not pause the other input groups**—keep paging the remaining inputs until they satisfy their goals or also reach their own price ceilings.
- Once the other input groups complete, revisit the blocked group and continue scanning its alternatives (same collection/rarity) page by page, still obeying the per-item price ceiling. Float constraints should continue to be ignored as stopping conditions, even across alternatives.
- Always log which inputs hit price ceilings, which are still searching, and the pages accessed, so the final summary can describe exactly what was found or why the trade-up cannot proceed.

#### 2d. Reporting when no matches are ever found

- If every input group (including all their alternatives) exhausts pages without reaching the goal and the only stop reason is a price ceiling, return a clear summary to the user that includes:
  1. Each input’s collected count vs needed.
  2. Pages accessed per input and whether the price ceiling was triggered.
  3. A recommendation (e.g., “Try a different trade-up or raise the budget”) if no inputs can be satisfied.
  4. Confirmation that float constraints were respected but never stopped the loops.

#### 2e. Listing fields

**Critical Enforcement Rules:**
- ✅ You MUST log state after every FetchFloatData call
- ✅ You MUST continue paging even if pages 1-5 return 0 results (float constraints take time)
- ✅ You MUST only stop when: (a) collected enough items, OR (b) price ceiling is hit across entire page
- ✅ You MUST update the state table in your response for the user to see pagination progress
- ✅ No page limit — keep paging until one of the two stop conditions is met

#### 2b. Pagination enforcement (MANDATORY COMPLIANCE)

**⚠️ CRITICAL: These are not guidelines — they are strict rules. Violating them means the skill is broken.**

**Rule 1: NO early abandonment based on float constraints**
- Pages 1-5 return 0 listings due to float being out of range → MUST continue (float distribution varies significantly)
- Float constraints are tight → This is EXPECTED. Keep paging; floats take time to find.
- **Exception:** ONLY stop if (a) you've collected enough, OR (b) price ceiling is hit on the current page

**Rule 2: Price ceiling is the only hard blocker**
- All listings on a page exceed maxPrice ceiling → STOP (prices only increase on later pages)
- A few listings exceed price but some pass → Continue (filter constraints apply at fetch time)
- If page returns 0 listings and prices don't hit ceiling → Continue paging

**Rule 3: No page limit**
- NO hard page limit (remove page 15 concept)
- Keep paging until one of the two stopping conditions: collected goal OR price ceiling hit
- This may take 25-50+ pages for tight float constraints — this is normal and expected

**Rule 4: State reporting is mandatory**
- AFTER every FetchFloatData call, update the pagination state table in your response
- Show: page number, listings found, progress toward goal
- Show: decision ("continue to page X" or "stop because Y")
- Do NOT suppress this output — users need to see pagination progress

**Rule 5: Stopping conditions (in order of priority)**
1. **Collected goal?** Collected ≥ input.Count items → Stop pagination
2. **Price ceiling hit?** FetchFloatData indicates all items exceed maxPrice → Stop pagination (no recovery possible)
3. **NO other conditions exist** → If neither above, continue paging (forever if needed)

**Red flags that indicate the skill was violated:**
- ❌ "I stopped at page 3 because floats were high" → VIOLATION (should continue)
- ❌ "I gave up because price exceeded ceiling on page 1" → VIOLATION (page 1 doesn't determine later pages for price)
- ❌ "I abandoned after page 5 with 0 results" → VIOLATION (should check pages 6-25+)
- ❌ "No pagination state table shown" → VIOLATION (state tracking is mandatory)
- ❌ "I stopped at page 15 for safety" → VIOLATION (no page limit exists)

#### 2e. Listing fields

Each listing has: `listingId`, `itemName`, `float`, `price`, `subtotalCents`, `feeCents`

#### 2d. Shortfall handling (systematic alternatives)

After primary item pagination exhausted (collected < needed):

```
collected = X (< input.Count)
alternatives = input.AlternativeSkins (ordered list, same collection/rarity)

FOR EACH alternative IN alternatives:
  IF (collected >= input.Count):
    → Input group satisfied. BREAK.

  collected_from_alt = pagination(alternative.itemName, alt.maxFloat, alt.maxPrice, expectedOutputValue: opportunity.bestOutput.Price)
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

### Step 2c — Merge subagent results (after parallel scouting completes)

Once all subagent pagination tasks complete, merge their results:

```
subagent_results = [task1_output, task2_output, ...]  # All completed tasks

merged_listings = {}  # key: input_group_id, value: list of listings

FOR EACH result IN subagent_results:
  item_name = result['item_name']
  listings = result['listings']  # Already filtered by fetch_float_data
  
  merged_listings[item_name] = listings
  
  ✓ Log: "Merged [len(listings)] listings for [item_name]"

# All listings are already float/price-filtered by the FetchFloatData calls in each subagent
# No additional filtering needed here
```

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

**Error handling for purchases:**

The purchase operation is a single call (not paginated). If it fails:

```
ATTEMPT 1: Call execute_trade_up_input_purchases with collected listings

❌ IF ERROR (any kind):
  ✓ Log the error details
  ✓ Apply exponential backoff: wait 1s, then 2s, then 4s (3 retries max)
  ✓ Each retry uses the same listings (operation is idempotent, no duplicate charges)
  ✓ After 3 retries fail: report failure to user with all listing IDs so they can diagnose or retry manually
    
    
    ✅ IF SUCCESS:
      ✓ Log all purchased items with prices, floats
      ✓ Report total spent and profit potential
    ```

    Report: items purchased, total spent, duration, any errors. If purchase partially succeeds (some items buy, some fail), report both success and failure items clearly.

---

## Important constraints

- **Filtering happens at FetchFloatData** — call FetchFloatData with `maxFloat` and `maxPrice` parameters to get pre-filtered listings from Steam. The service applies constraints at fetch time (inline filtering).
- **Float constraint** — pass the exact `maxFloat` from the opportunity (input.maxFloat) to FetchFloatData. This is a hard ceiling, not a relaxed tolerance. The opportunity specifies the maximum float to preserve the trade-up's profitability and hit probability.
- **Price constraint** — pass the exact `maxPrice` from the opportunity (input.maxPrice) to FetchFloatData. This is a hard ceiling pre-calculated by the opportunity to preserve profitability. Do not recalculate or adjust it.
- **Wiggle room calculation** — pass `expectedOutputValue: opportunity.bestOutput.Price` (the current market price of the output skin) to FetchFloatData for all scouting calls. The server uses this to calculate wiggle room: `wiggle = output_price - total_cost`. This margin buffer ensures the trade-up remains profitable even with small market fluctuations.
- **Pagination is mandatory and requires strict compliance** — You MUST follow the pagination loop exactly as defined in section 2a, including state tracking. This is not optional:
  - **Initialization:** Create and update a pagination state table showing page number, listings found, progress toward goal
  - **Every page:** Log your decision before and after the API call
  - **Page 1 = 0 results:** Continue to pages 2, 3, 4... (do NOT assume market is empty)
  - **Float constraints tight on page 1:** Continue paging — floats are random and widely distributed (normal to go 25-50+ pages)
  - **Prices high on page 1:** Continue paging if any items pass constraints (prices vary by page)
  - **Empty pages:** Keep paging unless price ceiling is hit (float distribution is natural variance)
  - **No page limit:** Remove the concept of page 15. Keep paging indefinitely.
  - **Stop conditions (only):** (a) Collected enough items, OR (b) Price ceiling hit (all items on page exceed maxPrice)
  - **Reporting:** Show pagination state table to user after each page (not just summaries)
- **Currency behavior** — all costs are in ZAR; the server stores and returns prices in ZAR.
- **Rate limit handling** — the server's FloatScraperService handles Steam rate limits internally (throttle delays, retry logic). If rate limited, FetchFloatData returns what it could fetch in the current request.

## Missing Info Policy

If any required field/behavior is unclear:
1. Inspect MCP tool schema and descriptions.
2. Run a minimal probe call with safe limits (`pageSize=1`) to infer response structure.
3. Continue the workflow using discovered fields.
4. Only ask the user when MCP discovery cannot resolve ambiguity.
