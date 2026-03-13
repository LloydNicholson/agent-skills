# Agent Skills

A curated collection of production-ready Claude Code skills for automated workflows.

## 📁 Skills Included

### 1. **CS2 Trade-Up Buyer** (`cs2-tradeup-buyer`)
Automated workflow for discovering profitable CS2 trade-up opportunities, scouting Steam Market listings, and executing purchases with budget/float constraints.

**Status:** Production-ready
**Files:**
- `SKILL.md` — Technical specification and implementation details
- `examples.md` — Copy-paste scenarios and quick reference
- `.instructions.md` — User guide and invocation syntax

**Quick start:** `/cs2-tradeup-buyer <maxCost>`

---

### 2. **GitHub PR Review** (`github-pr-review`)
Automated pull request review workflow with inline comments, approval/request-changes decisions, and summaries.

**Status:** Template (customize as needed)
**Files:**
- `SKILL.md` — Workflow specification

---

### 3. **Docker Debug** (`docker-debug`)
Diagnostic and troubleshooting skill for Docker containers and services.

**Status:** Template (customize as needed)
**Files:**
- `SKILL.md` — Workflow specification

---

## 🚀 Usage

Each skill is self-contained and portable. To use a skill:

1. **Copy the skill folder** to your local Claude Code skills directory:
   ```bash
   cp -r cs2-tradeup-buyer ~/.claude/skills/
   ```

2. **Invoke the skill** with the command specified in its `SKILL.md` or `.instructions.md`

3. **Reference the documentation** in each skill folder for detailed usage

## 📋 Skill Structure

```
<skill-name>/
├── SKILL.md              (Technical spec + implementation)
├── .instructions.md      (Optional: user-friendly guide)
├── examples.md           (Optional: scenarios and quick reference)
└── scripts/              (Optional: helper scripts)
```

## 🔧 Adding New Skills

To add a new skill to this repository:

1. Create a new folder at the root level
2. Add `SKILL.md` with frontmatter and technical specification
3. Add optional `.instructions.md` for user guidance
4. Add optional `examples.md` for common scenarios
5. Commit and push

## 📞 Support

For issues or improvements to individual skills, refer to the documentation within each skill folder.

---

**Last Updated:** 2026-03-13
**Repository:** Agent Skills
