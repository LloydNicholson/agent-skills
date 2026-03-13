# Skill Loader Pattern

A portable system that lets agents automatically discover and load skills from multiple sources without hardcoding paths.

## Problem This Solves

Without a loader, agents have to know:
- Exact file paths to skills
- Which repos have which skills
- Different paths on different machines
- Different MCP tool names across providers

With a loader, agents just ask: *"What skills do I have?"* and the system returns all available skills.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Agent asks: "Load all available skills"                      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Skill Loader searches in order:                              │
│  1. .skills/ (project local)                                 │
│  2. ~/.agent-skills/ (user home)                             │
│  3. ~/ai/skills/skills/ (global)                             │
│  4. MCP registry (if configured)                             │
│  5. GitHub search (if enabled)                               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Returns metadata about all found skills:                      │
│  - name, description, location, type, version                │
│  - SKILL.md path, trigger patterns, dependencies             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Agent loads selected skills and executes                     │
└─────────────────────────────────────────────────────────────┘
```

## Implementation

### 1. Skill Registry File (SKILL.md Frontmatter)

Each skill starts with YAML frontmatter:

```yaml
---
name: cs2-tradeup-buyer
description: Find and execute CS2 trade-up purchases
version: 1.0.0
triggers:
  - /cs2-tradeup-buyer <maxCost>
  - "find and buy tradeup"
dependencies: []
tags: ["cs2", "trading", "automation"]
---
```

### 2. Skill Index (skills.json)

Store in `~/ai/skills/` or `.skills/`:

```json
{
  "skills": [
    {
      "name": "cs2-tradeup-buyer",
      "path": "skills/cs2-tradeup-buyer",
      "version": "1.0.0",
      "source": "local",
      "triggers": [
        "/cs2-tradeup-buyer <maxCost>",
        "find and buy tradeup"
      ]
    },
    {
      "name": "github-pr-review",
      "path": "skills/github-pr-review",
      "version": "0.1.0",
      "source": "local",
      "triggers": [
        "/github-pr-review <PR_URL>"
      ]
    }
  ],
  "version": "1.0"
}
```

### 3. Loader Script (Python Example)

```python
import os
import json
import yaml
from pathlib import Path
from typing import List, Dict

class SkillLoader:
    def __init__(self):
        self.search_paths = [
            Path.cwd() / ".skills",           # Project local
            Path.home() / ".agent-skills",    # User home
            Path("/Users/lloyd/ai/skills/skills"),  # Global
        ]

    def load_all_skills(self) -> List[Dict]:
        """Discover all available skills."""
        skills = []

        for base_path in self.search_paths:
            if not base_path.exists():
                continue

            # Look for SKILL.md files
            for skill_dir in base_path.glob("*/"):
                skill_md = skill_dir / "SKILL.md"
                if skill_md.exists():
                    skill_data = self._parse_skill(skill_md, skill_dir)
                    skills.append(skill_data)

        return skills

    def _parse_skill(self, skill_md_path: Path, skill_dir: Path) -> Dict:
        """Parse SKILL.md frontmatter and metadata."""
        with open(skill_md_path, "r") as f:
            content = f.read()

        # Extract frontmatter
        if content.startswith("---"):
            parts = content.split("---", 2)
            frontmatter = yaml.safe_load(parts[1])
        else:
            frontmatter = {}

        return {
            "name": frontmatter.get("name", skill_dir.name),
            "description": frontmatter.get("description", ""),
            "version": frontmatter.get("version", "0.1.0"),
            "path": str(skill_dir),
            "skill_md": str(skill_md_path),
            "triggers": frontmatter.get("triggers", []),
            "dependencies": frontmatter.get("dependencies", []),
            "tags": frontmatter.get("tags", []),
        }

    def get_skill(self, name: str) -> Dict:
        """Get a specific skill by name."""
        all_skills = self.load_all_skills()
        for skill in all_skills:
            if skill["name"] == name:
                return skill
        raise ValueError(f"Skill '{name}' not found")

    def find_by_trigger(self, trigger: str) -> List[Dict]:
        """Find skills matching a trigger pattern."""
        all_skills = self.load_all_skills()
        matches = []
        for skill in all_skills:
            for skill_trigger in skill.get("triggers", []):
                if trigger.lower() in skill_trigger.lower():
                    matches.append(skill)
                    break
        return matches

# Usage
if __name__ == "__main__":
    loader = SkillLoader()

    # List all skills
    all_skills = loader.load_all_skills()
    for skill in all_skills:
        print(f"✓ {skill['name']} ({skill['version']})")
        print(f"  Path: {skill['path']}")
        print(f"  Triggers: {', '.join(skill['triggers'][:2])}")
        print()

    # Find a specific skill
    cs2_skill = loader.get_skill("cs2-tradeup-buyer")
    print(f"Loaded: {cs2_skill['skill_md']}")
```

### 4. Agent Integration Pattern

```python
# In your agent code
from skill_loader import SkillLoader

loader = SkillLoader()

# Option A: Load specific skill
skill = loader.get_skill("cs2-tradeup-buyer")
with open(skill["skill_md"]) as f:
    instructions = f.read()
# Agent now has the full SKILL.md content to follow

# Option B: Find by trigger
matching_skills = loader.find_by_trigger("trade-up")
# Returns all skills with "trade-up" in triggers

# Option C: List all available
all_skills = loader.load_all_skills()
print(f"Agent has {len(all_skills)} skills available")
```

## Search Order (Priority)

Skills are discovered in this order (first match wins):

1. **`.skills/`** (project-local) — highest priority
2. **`~/.agent-skills/`** (user home)
3. **`~/ai/skills/skills/`** (global)
4. **MCP registry** (if configured)
5. **GitHub** (if enabled with token)

This means:
- Project can override global skills
- User can override global with home folder
- Global is fallback for all machines

## Advantages

✅ **Zero hardcoding** — No absolute paths in agent code
✅ **Works offline** — Doesn't need network unless GitHub enabled
✅ **Multi-repo** — Agents find skills from multiple sources
✅ **Versioning** — Skills tagged and indexed by version
✅ **Discovery** — Agents self-aware of available skills
✅ **Portable** — Works across machines and projects
✅ **Extensible** — Can add new search sources easily

## Example: Using in Claude Code

When your agent (Claude) starts, it could:

1. Run `skill_loader.py` to get available skills
2. Read the frontmatter from relevant SKILL.md files
3. Load instructions into context
4. Execute with full awareness of available capabilities

## Optional: Central Index Server

For teams, you could host a skill index server:

```
GET /api/skills
→ Returns JSON of all available skills with metadata

GET /api/skills/cs2-tradeup-buyer
→ Returns full SKILL.md + examples.md + version history

GET /api/skills/search?tag=trading
→ Returns all trading-related skills
```

This lets:
- Multiple machines sync skill metadata
- Agents query a canonical registry
- Version control at the registry level
- Analytics on skill usage

---

## Summary

The Skill Loader pattern means:
- **Agents don't care where skills are stored**
- **Skills are discoverable, versioned, and portable**
- **Works with any backend (local, GitHub, MCP, custom servers)**
- **Simple to implement, powerful in practice**
