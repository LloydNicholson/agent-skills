# Examples & Quick Reference

Copy-paste ready examples for common CS2 trade-up scouting scenarios.

## Example 1: Budget Trade-Up (under R20)

**User Request:**
```
Find the most profitable trade-up under 20 Rand with unique outputs, sorted by profitability
```

**Invocation:**
```
/cs2-tradeup-buyer 20 unique
```

**Expected Timeline:**
- Search: 2–5 seconds
- Scout primary skins: 1–2 minutes
- Potentially try alternatives: +1–2 minutes
- Confirmation & purchase: 2–5 minutes
- **Total: 5–10 minutes**

**What You'll See:**
```
✅ Found 8 opportunities under R20 (unique outputs)

📊 SELECTED: Trade-Up #24769268
   Output: Glock-18 | Green Line (Factory New)  
   Profit: 238% (R71.80 profit on R30.16 spend)
   Type: Multi
   
   INPUT 1: 3× CZ75-Auto | Copper Fiber (Minimal Wear) @ R2.36/ea
   INPUT 2: 7× MP5-SD | Neon Squeezer (Field-Tested) @ R3.30/ea

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

SCOUTING MARKET...

INPUT 1 (need 3):
  ✓ CZ75-Auto | Copper Fiber: 2 found (float 0.0789–0.0810, price R2.35–R2.40)
  ✓ P90 | Straight Dimes [ALT]: 1 found (float 0.0790, price R2.10)
  ✅ COMPLETE (3/3)

INPUT 2 (need 7):
  ✓ MP5-SD | Neon Squeezer: 5 found
  ✓ Glock-18 | Graveyard [ALT]: 2 found
  ✅ COMPLETE (7/7)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

PURCHASE SUMMARY:
✅ 10 items located
   Total Cost: R30.16
   Profit Potential: R71.80 (238%)

Ready to purchase? (yes/no)
```

## Example 2: Second-Best Trade-Up (Diversify)

**User Request:**
```
I want the second most profitable trade-up under R100, with a different output item than the last one
```

**Invocation:**
```
/cs2-tradeup-buyer 100 second unique
```

**Result:**
- Searches opportunities, **skips #1**, returns **#2 ranked**
- Guarantees different output (unique filter)
- Proceeds with scouting and purchase if confirmed

---

## Example 3: High-Profit Threshold

**User Request:**
```
Find any trade-up under R50 that's at least 150% profit, with unique outputs
```

**Invocation:**
```
/cs2-tradeup-buyer 50 minProfit=150 unique
```

**Expected:** Only opportunities with 150%+ profit margin shown.

---

## Example 4: USD Display

**User Request:**
```
Take-up under R200, but show prices in USD
```

**Invocation:**
```
/cs2-tradeup-buyer 200 currency=USD
```

**Result:**
- All prices displayed in USD instead of ZAR
- Profit calculations in USD
- Backend auto-converts using current rates

---

## Quick Reference: Common Filters

| Goal | Command |
|------|---------|
| Cheapest trade-up | `/cs2-tradeup-buyer 10` |
| Most profitable under budget | `/cs2-tradeup-buyer 50 unique` |
| High-profit only | `/cs2-tradeup-buyer 100 minProfit=50` |
| Diversify outputs | `/cs2-tradeup-buyer 75 second unique` |
| Large budget | `/cs2-tradeup-buyer 500` |
| USD pricing | `/cs2-tradeup-buyer 50 currency=USD` |
| Third-ranked option | `/cs2-tradeup-buyer 100 third unique` |

---

## Expected Performance

### Market Scouting Times
- **Fast (< 3 min):** Primary skin available, few inputs
- **Moderate (3–5 min):** Need 1–2 alternatives per input
- **Slow (5–10 min):** Many alternatives, multiple inputs
- **Very slow (10+ min):** Rare items, many alternatives, high count

### Purchase Times
- **Fast (1–2 min):** 5–10 items, all listed immediately
- **Moderate (2–5 min):** 10+ items, standard delays
- **Slow (5–10 min):** 20+ items, rate limit pauses

### When It Stops
- ⏹️ **Rate limit hit** → Immediate abort, report shortfall
- ❌ **0 items found for any input** → Fails (can't execute partial trade-up)
- ⚠️ **Shortfall (7 of 10)** → Asks user to confirm or retry

---

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| "Rate limit hit" on first fetch | Account used recently for market browsing | Wait 30+ min, try again |
| Partial shortfall (8 of 10) | Rare item, exhausted market pages | Accept partial or try different trade-up |
| Slow scouting (10+ min) | Many alternatives, large item count | Normal for difficult trade-ups |
| "0 items found" | Market inventory depleted or float spec too tight | Relax tolerances (automatic) or abort |
| Purchase fails with confirmation | Market inventory changed during delay | Rerun scout to get fresh listings |
| "-32001 Session not found" error | MCP server session timed out (3–5 min idle) | See below → Auto-recovery |

### Session Timeout Recovery (-32001 Error)

**What it is:** The MCP server closes sessions after ~3–5 minutes of inactivity. This can happen during pagination (scouting) or purchase.

**What the skill does automatically:**
1. Detects the error
2. Reloads the skill (creates a new MCP session)
3. Restores the last checkpoint (page number, items collected)
4. Retries from that checkpoint
5. Logs the recovery for transparency

**Example: Pagination gets interrupted**
```
📊 Page 3 of MP9 | Black Sand — 2/10 items collected
❌ Session timeout on page 3
🔄 Reloading skill...
✅ Skill reloaded. New session established.
📍 Checkpoint found: page 3, 2 items collected
🔄 Resuming from page 3...
✅ Page 3: 3 more items found
📊 Progress: 5/10 collected. Continuing...
```

**Example: Purchase gets interrupted**
```
💳 Executing purchase of 10 items for R25.86...
❌ Session timeout during purchase
🔄 Reloading skill...
✅ Skill reloaded. New session established.
🔄 Retrying purchase (attempt 2)...
✅ All 10 items purchased successfully
```

**If recovery fails after 3 attempts:**
```
❌ Purchase failed after 3 retry attempts
📋 Items ready to buy (collected during scouting):
   - 3× MP9 | Black Sand (R2.51 each)
   - 7× AUG | Commando Company (R2.13 each)
💡 Try again in a few moments — service may recover
```

**What you need to do:** Nothing! The recovery is automatic. If you see checkpoint restore messages, the skill is handling the timeout. Just wait for it to complete.

---

## Advanced: Rate Limit Strategy

The skill uses **2-second delays** between all Steam Market fetches. Here's why:

```
Normal Account (Authenticated)
├─ 1–2 requests/sec → 3600 req/hour max
├─ 3600 / 60 = 60 req/min
└─ Hit limit in ~20 minutes ❌

Anonymous Proxy (This Skill)
├─ Proxy IP = ~100–200 requests/sec allowed
├─ 2-second delay = 30 req/min = 1800 req/hour
├─ Only 1800 / 100 = 18% of proxy quota
└─ Safe for hours of continuous scouting ✅
```

**Key Safety Rules:**
- 🛑 **Always stop immediately on rate limit** — no retries
- ⏱️ **Never remove the 2-second delay** — prevents cascade failures
- 🌍 **Anonymous proxy traffic only** — no cookies, no auth
- 🔄 **Serial fetching only** — never parallel requests

