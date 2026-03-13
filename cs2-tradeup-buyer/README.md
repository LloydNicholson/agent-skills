# CS2 Trade-Up Buyer Skill

A complete, production-ready MCP skill for automated CS2 trade-up opportunity discovery, market scouting, and purchase execution.

## 📁 Skill Package Contents

```
/cs2-tradeup-buyer/
├── .instructions.md        ← START HERE (user guide + quick ref)
├── SKILL.md               (technical spec + implementation details)
├── EXAMPLES.md            (copy-paste scenarios + reference)
├── README.md              (this file)
└── (no dependencies — fully self-contained)
```

## 🚀 Quick Start

**Read this first:** [.instructions.md](.instructions.md)

**Then use:** `/cs2-tradeup-buyer <maxCost> [options]`

**Example:** `/cs2-tradeup-buyer 50 unique`

## 📖 File Guide

| File | Purpose | For Whom |
|------|---------|----------|
| **.instructions.md** | User-friendly guide, invocation syntax, examples, expected behavior | Everyone — read first |
| **SKILL.md** | Technical workflow, rate limit strategy, float/price tolerance rules | Developers, advanced users |
| **EXAMPLES.md** | Copy-paste scenarios, quick reference table, troubleshooting | Users implementing specific scenarios |
| **README.md** | This index (you are here) | Navigation |

## ✨ Key Features

- ✅ **Fully Automated** — One command for entire scouting + purchase workflow
- ✅ **Rate Limit Safe** — 2-second delays, anonymous proxy, immediate abort strategy
- ✅ **Unlimited Alternatives** — Tries ALL same-collection/rarity substitutes (no hard limits)
- ✅ **Partial Success OK** — Continues even if 1–2 items fail to purchase
- ✅ **No Authentication Required** — Unauthenticated market fetches (safer, faster)
- ✅ **User Confirmation** — Shows cost breakdown before purchase
- ✅ **Clear Reporting** — Individual item status, shortfalls, profit potential

## 🎯 Workflow at a Glance

1. **Discover** — Find profitable opportunities within budget
2. **Scout** — Fetch real-time float data + prices from Steam Market
3. **Validate** — Compare against spec (price/float tolerances)
4. **Confirm** — Show summary, get user approval
5. **Execute** — Purchase items with partial-success handling

**Total time:** 5–15 minutes depending on market availability.

## 🛡️ Safety Guarantees

- 🔒 **No account damage** — Anonymous proxy, no authentication
- ⏸️ **Rate limit protection** — Stops immediately if Steam throttles
- 📊 **Transparent** — Reports exactly what was found/purchased
- ✋ **User control** — Always asks for confirmation before buying

## 🔌 Integration

The skill connects to the **CS2MarketBot MCP Server** backend via:
- **get_trade_up_opportunities** — Find opportunities
- **fetch_float_data** — Get market listings with float values  
- **execute_trade_up_input_purchases** — Execute purchases

All credentials and API access handled server-side (no client setup needed).

## 📋 Parameters

```
/cs2-tradeup-buyer <maxCost> [unique] [rank] [currency] [minProfit]

Examples:
  /cs2-tradeup-buyer 20              (R20 max, most profitable)
  /cs2-tradeup-buyer 50 unique       (unique outputs, ranked)
  /cs2-tradeup-buyer 100 second      (2nd most profitable)
  /cs2-tradeup-buyer 75 currency=USD (prices in USD)
```

See [.instructions.md](.instructions.md) for full parameter reference.

## 🔧 Configuration & Customization

**Default Settings:**
- **Rate limit delay:** 2 seconds (per anonymous proxy limits)
- **Page size:** 100 listings per fetch (Steam Market max)
- **Price tolerance:** ±10% over spec
- **Float tolerance:** ±0.01 above spec
- **Alternative limit:** Unlimited (all same-rarity items tried)

**To modify:** Edit `SKILL.md` Step 2a/2b or [FloatScraperService.cs](../../src/CS2MarketBot.Services/FloatScraperService.cs#L19) for backend throttle delay.

## 📚 Documentation Structure

```
Quick Start Path:
  Read .instructions.md (5–10 min) → Run command → Done

Advanced Path:
  Read .instructions.md → Check EXAMPLES.md → Review SKILL.md

Troubleshooting Path:
  EXAMPLES.md → Troubleshooting section → SKILL.md (if needed)

Implementation Path:
  SKILL.md (full spec) → EXAMPLES.md (scenarios) → Deploy
```

## ✅ Checklist: Is This Skill Ready?

- [x] Complete workflow documented (discovery → purchase)
- [x] Rate limit strategy clearly explained (2-second delays, anonymous proxy)
- [x] Examples & quick reference provided
- [x] Safety guarantees documented (no account damage, immediate abort)
- [x] User confirmation required before purchase
- [x] Partial purchase success handled correctly
- [x] All alternatives (unlimited) are tried
- [x] Unauthenticated float fetching (no cookies)
- [x] Backend integration working (MCP tools available)
- [x] Tested & deployed to production

## 🐛 Known Limitations

- **Serial scouting only** — No parallel fetches (by design, for rate limit safety)
- **Takes 5–15 min** — Trade-up scouting is not instant due to deliberate delays
- **Shows shortfalls clearly** — If 0 items found for any input, trade-up cannot proceed
- **Manual confirmation required** — For safety; always review before purchase

These are **intentional constraints** for safety and reliability.

## 📞 Support

**Issue:** Stuck / confused about how to use?  
**Solution:** Read [.instructions.md](.instructions.md) → Look up your scenario in [EXAMPLES.md](EXAMPLES.md)

**Issue:** Want to change rate limits?  
**Solution:** See [SKILL.md](SKILL.md) Step 2a — Document explains delay strategy

**Issue:** Command not working?  
**Solution:** Check [EXAMPLES.md](EXAMPLES.md) troubleshooting section

---

**Version:** 1.0 (Production)  
**Last Updated:** 2026-03-12  
**Status:** Ready to use — all components integrated and tested
