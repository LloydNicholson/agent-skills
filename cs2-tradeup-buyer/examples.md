# Examples & Quick Reference

Use `cli-anything-cs2marketbot` for discovery, scouting, execution, and auth checks.

## Example 1: Budget search

```bash
cli-anything-cs2marketbot --json opportunities list --max-cost 20 --unique-by-output --sort profit
```

Expected outcome:
- JSON list of opportunities
- Select the best candidate for the user’s budget

## Example 2: Scout a candidate

```bash
cli-anything-cs2marketbot --json execute scout 27563035
```

Then poll status:

```bash
cli-anything-cs2marketbot --json execute status <job-id>
```

## Example 3: Check auth state

```bash
cli-anything-cs2marketbot --json auth status
```

## Example 4: Set the default server URL

```bash
cli-anything-cs2marketbot session config --set-url https://cs2marketbot.lloydindevelopment.work
```

## Example 5: Query a price

```bash
cli-anything-cs2marketbot --json price get "AK-47 | Redline" --wear FT
```

## Quick reference

| Goal | Command |
|------|---------|
| Find profitable opportunities | `cli-anything-cs2marketbot --json opportunities list --max-cost 20 --sort profit` |
| Limit to unique outputs | add `--unique-by-output` |
| Scout an opportunity | `cli-anything-cs2marketbot --json execute scout <id>` |
| Start purchase | `cli-anything-cs2marketbot --json execute start <id>` |
| Poll progress | `cli-anything-cs2marketbot --json execute status <job-id>` |
| Check auth | `cli-anything-cs2marketbot --json auth status` |
| Save default URL | `cli-anything-cs2marketbot session config --set-url <url>` |

## Notes

- Keep the workflow serial: one candidate at a time.
- Always summarize the scout before asking for purchase confirmation.
- Use `--json` whenever you need to parse the result in code.
