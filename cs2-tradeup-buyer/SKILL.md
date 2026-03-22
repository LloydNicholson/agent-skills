---
name: cs2-tradeup-buyer
description: >
  CLI-driven workflow for finding, scouting, and executing profitable CS2 trade-ups
  with cli-anything-cs2marketbot.
---

# CS2 Trade-Up Buyer (CLI)

Use the installed `cli-anything-cs2marketbot` command to discover opportunities, scout listings, confirm purchases, and monitor execution status.

## Core rule

Do not call the web app API directly. Always go through the CLI that was just built:

```bash
cli-anything-cs2marketbot --help
```

If the server URL needs to be set, use:

```bash
cli-anything-cs2marketbot session config --set-url https://cs2marketbot.lloydindevelopment.work
```

The CLI stores config locally and also accepts `--base-url` when needed.

## Supported workflow

1. Read the user’s budget and any filters.
2. List opportunities with the CLI.
3. Pick the best candidate from the returned JSON.
4. Scout the candidate with the CLI.
5. Poll execution status until the scout completes.
6. Present the scout results and ask for confirmation.
7. If the user approves, start the full execution with the CLI.
8. Poll until the purchase finishes.
9. Summarize the final result.

## Primary commands

### Discover opportunities

```bash
cli-anything-cs2marketbot --json opportunities list   --max-cost 20   --unique-by-output   --sort profit   --page-size 5
```

Useful filters:
- `--min-profit`
- `--min-difficulty`
- `--max-difficulty`
- `--min-cost`
- `--max-cost`
- `--min-hit-chance`
- `--stattrak`
- `--last-hours`
- `--type`
- `--sort`
- `--page`
- `--page-size`
- `--unique-by-output`
- `--currency`

### Scout and execute

```bash
cli-anything-cs2marketbot --json execute scout <opportunity-id>
cli-anything-cs2marketbot --json execute status <job-id>
cli-anything-cs2marketbot --json execute start <opportunity-id>
```

### Supporting commands

```bash
cli-anything-cs2marketbot --json auth status
cli-anything-cs2marketbot --json price get "AK-47 | Redline" --wear FT
cli-anything-cs2marketbot session config
```

## Decision policy

- If no opportunities are returned, ask the user to adjust budget or filters.
- If scouting reports a shortfall, show the exact shortfall and ask whether to try a different opportunity.
- If scouting succeeds, always ask for confirmation before starting the purchase phase.
- If the CLI exits non-zero, inspect stderr and retry only if the error looks transient.
- Prefer `--json` for every machine-readable step.

## Response format

When reporting results to the user, show:
- selected opportunity ID
- output skin
- total cost
- profit percent
- profit chance
- any input shortfalls
- final purchase/job status

Keep the workflow serial: one candidate at a time, one scout at a time, one purchase decision at a time.
