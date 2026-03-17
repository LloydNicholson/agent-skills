---
name: cs2-tradeup-buyer
description: >
  REST API-driven workflow to scout and monitor CS2 trade-up opportunities.
  Use this skill whenever the user wants to find profitable trade-ups, scout inputs, and optionally execute purchases.
  Triggers on: /cs2-tradeup-buyer <maxCost>, "scout tradeup under R<X>", "find and execute tradeup", 
  "check tradeup opportunities", or any request to find and monitor trade-up opportunities.
---

# CS2 Trade-Up Buyer (Watcher)

Automates the full workflow: find a target trade-up opportunity within budget constraints → submit scout-only job via API → poll execution status → ask for purchase confirmation → submit execution job via API.

## Portability Guarantees

- Do not rely on local repository files or source code.
- Operate only through REST API endpoints at the configured base URL.
- Do not require users to provide credentials; assume backend handles authentication.
- If the API base URL is not configured, ask the user to provide it.

## Configuration

**API Base URL:** The skill requires a base URL for the REST API (default: infer from environment or ask user).

Example: `https://cs2marketbot.lloydindevelopment.work/api`

All endpoints are relative to this base URL.

## Invocation

```
/cs2-tradeup-buyer <maxCost>
```

`maxCost` is the maximum total cost in ZAR (e.g., `20` = R20). If omitted, ask the user for it.

Optional intent examples:
- "scout tradeup opportunities under R10"
- "find most profitable tradeup under R50"
- "execute tradeup under R25"

---

## Workflow

### Overview

The skill executes this deterministic loop:
1. Parse user input → get parameters (budget, filters)
2. Call `GET /api/trade-ups/opportunities` → find best trade-up
3. Call `POST /api/trade-ups/{id}/execute?scoutOnly=true` → start async scout job
4. Poll `GET /api/trade-ups/executions/{jobId}` until complete
5. Display results (found items, costs)
6. Ask user for confirmation
7. Call `POST /api/trade-ups/{id}/execute?scoutOnly=false` → start purchase job (if approved)
8. Poll execution status until complete
9. Report results

**Key principle:** The skill coordinates the workflow via REST API. The API handles all scouting, market fetching, and purchase logic internally.

---

## REST API Endpoints
- `GET /api/trade-ups/opportunities` — List available trade-up opportunities
- `POST /api/trade-ups/{id}/execute?scoutOnly=true` — Scout-only execution (no purchases)
- `POST /api/trade-ups/{id}/execute?scoutOnly=false` — Execute with purchases
- `GET /api/trade-ups/executions/{jobId}` — Poll execution status
- `GET /api/trade-ups/executions` — List recent executions

**Async job pattern:**
- All executions return a `jobId`.
- Poll status via API until complete.
- If purchase is required, ask user for confirmation before submitting purchase job.

**Scout-only parameter:**
- Use `scoutOnly=true` for scout-only (no purchases).
- Use `scoutOnly=false` for full execution (with purchases).

---

## Step-by-Step Workflow

### Step 1 — Parse user input

Extract these parameters:
- `maxCost` (required) — User's budget
- `minProfit` (optional) — Minimum profit % (default: no minimum)
- `tradeUpType` (optional) — Filter by type (default: all)
- `currency` (optional) — Display currency (default: ZAR)

### Step 2 — Fetch opportunities

Call `GET /api/trade-ups/opportunities` with:
- `maxCost` = user budget
- `sortBy` = "profitPercent" (to rank by profit %)
- `pageSize` = 5
- Add optional filters based on user input

Display the top opportunity:
```
✅ Found 12 opportunities under R50
📊 Trade-Up #24769268 — Glock-18 | Green Line (Factory New)
   Type: Multi | Profit: 238% | Cost: R30.16
   
   INPUT 1: 3× CZ75-Auto | Copper Fiber (Minimal Wear)
   INPUT 2: 7× MP5-SD | Neon Squeezer (Field-Tested)
```

If no opportunities found, stop and ask user to adjust criteria (e.g., increase budget).

### Step 3 — Scout via API

Call `POST /api/trade-ups/{id}/execute?scoutOnly=true` with the selected opportunity ID.

**Response:** `{ jobId: 12345, status: "Pending", ... }`

### Step 4 — Poll until scout complete

Loop: Call `GET /api/trade-ups/executions/{jobId}` every 2-3 seconds until `status` is "Completed" or "Failed".

Log progress:
```
🔄 Scouting... (job 12345)
   Page 1 ✓ | Fetched float data...
   Page 2 ✓ | Collecting alternatives...
```

When complete, parse `inputsScouted`:
- For each input, display: name, needed, found, cost
- Calculate total cost
- Check if all inputs were found (status == "Complete" and anyFailed == false)

### Step 5 — Report scout results

If all inputs found:
```
✅ SCOUT SUCCESSFUL

INPUT 1 — CZ75-Auto | Copper Fiber (Minimal Wear)
  Need: 3 | Found: 3 ✅
  Cost: R7.44
  
INPUT 2 — MP5-SD | Neon Squeezer (Field-Tested)
  Need: 7 | Found: 7 ✅
  Cost: R22.72

═══════════════════════════════════════════
Total Cost: R30.16
Expected Output Price: R102.00
Profit Potential: R71.80 (238%)

Proceed with purchase? (yes/no)
```

If any inputs were NOT found (needed > found):
```
⚠️ PARTIAL SCOUT

INPUT 1 — CZ75-Auto | Copper Fiber (Minimal Wear)
  Need: 3 | Found: 2 ⚠️
  Cost: R4.96
  
INPUT 2 — MP5-SD | Neon Squeezer (Field-Tested)
  Need: 7 | Found: 7 ✅
  Cost: R22.72

═══════════════════════════════════════════
Total Cost: R27.68 (R2.48 under budget)

⚠️ One input is short. Proceed with partial execution? (yes/no/try-different)

Options:
- "yes" — Proceed with 9 items (2 short)
- "no" — Abort
- "try-different" — Try next best opportunity
```

If status is "Failed", report the error and ask if user wants to retry or try a different opportunity.

### Step 6 — Execute (if approved)

If user confirms, call `POST /api/trade-ups/{id}/execute?scoutOnly=false` with the same opportunity ID.

**Response:** `{ jobId: 54321, status: "Pending", ... }`

### Step 7 — Poll until purchase complete

Loop: Call `GET /api/trade-ups/executions/{jobId}` every 2-3 seconds until complete.

Log progress:
```
💳 Purchasing... (job 54321)
   Item 1 ✓ | Item 2 ✓ | Item 3 ✓ ...
```

When complete, parse `executionDetails`:
- `totalItemsFound` — How many items the API located
- `totalCost` — Final cost
- `itemsPurchased` — Success count
- `itemsFailed` — Failure count

### Step 8 — Report final results

```
✅ PURCHASE COMPLETE

Items Purchased: 10/10
Total Spent: R30.16
Duration: 2.5 minutes

📊 Profit Potential: R71.80 (238%)

Trade-up successful! Monitor your inventory for the output skin.
```

Or if partial:
```
⚠️ PURCHASE PARTIAL

Items Purchased: 9/10
Items Failed: 1/10
Total Spent: R27.68

⚠️ Trade-up may fail due to missing input. Check your inventory.
```

Or if purchase failed:
```
❌ PURCHASE FAILED

Reason: [error details from API]
Items Found: 10
Items Purchased: 0

Check API status or try again later.
```

---

## Error Handling

### API Errors

If any API call returns an error (4xx or 5xx):

1. **Log the error** with request details
2. **Apply exponential backoff** (1s → 2s → 4s) and retry up to 3 times
3. **Report to user** with collected data if operation was interrupted
4. **Ask** if user wants to retry or abort

Example:
```
❌ API Error (500): Server error on /api/trade-ups/opportunities

Retrying in 1 second...
Retrying in 2 seconds...
Retrying in 4 seconds...

After 3 retries, still failing. Please check API status and try again.
```

### Polling Timeout

If polling exceeds 10 minutes without completion, report timeout and suggest checking API status or manual inspection.

### Missing Opportunities

If no opportunities found for the given budget:
```
❌ No trade-up opportunities found under R[maxCost]

Try:
- Increase budget (e.g., /cs2-tradeup-buyer 50)
- Remove profit filters
- Check back later (opportunities update daily at ~06:00 SAST)
```

### Partial Execution

If the API completes with some items purchased and some failed, report both:
```
⚠️ PARTIAL EXECUTION

Purchased: 9/10 items
Failed: 1/10 item

Reason for failure: [per-item error if available]

Your trade-up may not complete. Check your inventory and consider retrying the failed item(s) manually.
```

---

## Important Notes

- **No internal scouting.** The API handles all market fetching, float filtering, and price constraints. Your role is to trigger jobs and poll results.
- **Stateless design.** The API is stateless; session timeouts do not occur. If polling fails, retry the request.
- **Polling is mandatory.** Always poll until completion; do not assume a job completes immediately.
- **User confirmation.** Always ask user before executing (non-scout) purchases.
- **Currency.** All prices are in ZAR; the API converts if other currency is requested.
- **Idempotent operations.** Calling execute twice with the same ID is safe (no duplicate charges).

