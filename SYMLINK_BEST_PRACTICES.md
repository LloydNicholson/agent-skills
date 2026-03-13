# Best Practices: Project .skills Symlinks

How to properly set up `.skills` symlinks in projects so agents automatically discover skills.

## Overview

Instead of agents hunting for skills in multiple locations, every project has a `.skills/` symlink that points to your global skill registry.

```
my-project/
├── .skills -> ~/ai/skills/skills    ← symlink
├── src/
├── tests/
└── README.md

# When agent scans my-project, it finds:
.skills/cs2-tradeup-buyer/SKILL.md
.skills/github-pr-review/SKILL.md
.skills/docker-debug/SKILL.md
```

## Setup

### Single Project

```bash
cd my-project
ln -s ~/ai/skills/skills .skills
git add .gitignore  # (see below)
```

### All Existing Projects

```bash
# Create symlinks in all project directories
for dir in ~/projects/*/; do
  ln -s ~/ai/skills/skills "$dir/.skills" 2>/dev/null
done

# Verify
find ~/projects -name ".skills" -type l
```

### Automation (Add to Login Script)

```bash
# ~/.zshrc or ~/.bashrc

# Ensure .skills symlinks exist in all project dirs
sync-project-skills() {
  local skills_path="$HOME/ai/skills/skills"
  find $HOME/projects -maxdepth 1 -type d ! -name "." | while read dir; do
    if [ ! -L "$dir/.skills" ]; then
      ln -s "$skills_path" "$dir/.skills"
      echo "✓ Linked $dir/.skills"
    fi
  done
}

# Run on login
sync-project-skills
```

## Git Configuration

### Option 1: Commit Symlink to Git (Recommended)

```bash
# .gitignore - exclude skill implementations
skills/

# But allow the symlink itself
!.skills
```

This way:
- Symlink is versioned
- Team members get the same symlink
- Git tracks `.skills -> ~/ai/skills/skills`

```bash
cd my-project
git add .gitignore
git commit -m "Add .skills symlink to project"
```

### Option 2: Symlink Not in Git

If you prefer not to version the symlink:

```bash
# .gitignore
.skills/

# .gitattributes (optional, documents expectation)
.skills symlink=true
```

Then each developer runs:
```bash
ln -s ~/ai/skills/skills .skills
```

**I recommend Option 1** — commit the symlink so the expectation is automatic.

## Handling Merge Conflicts

If two branches both create `.skills` symlinks:

```bash
# Git will show it as a regular file conflict
# Resolve by ensuring both point to the same target:

git rm .skills              # Remove the "file"
ln -s ~/ai/skills/skills .skills  # Recreate symlink
git add .skills
git commit -m "Resolve .skills symlink"
```

## Breaking Changes: What to Do When Skills Change

### Scenario: You update a skill that breaks existing projects

**Approach 1: Semantic Versioning (Recommended)**

```
~/ai/skills/
├── v1/skills/
│   ├── cs2-tradeup-buyer/ (v1.0)
│   └── docker-debug/
├── v2/skills/
│   ├── cs2-tradeup-buyer/ (v2.0 - breaking change)
│   └── docker-debug/
└── skills -> v1/skills    (default symlink points to v1)
```

Projects can opt-in to v2:
```bash
# For projects ready for v2
ln -sf ~/ai/skills/v2/skills .skills
```

**Approach 2: Gradual Migration**

```bash
# old-project/.skills -> ~/ai/skills/skills
# new-project/.skills -> ~/ai/skills/v2/skills
```

**Approach 3: Git Tags (If skills are a repo)**

```bash
cd ~/ai/skills
git tag v1.0.0
# Update skills...
git tag v2.0.0

# Check out specific version in projects
cd my-project
ln -s ~/ai/skills/skills-v1.0.0 .skills
```

## Scoping: When to Add Project-Specific Skills

Your project might have skills that only apply to that project.

```
my-project/
├── .skills -> ~/ai/skills/skills   (global)
├── .project-skills/                (project-local)
│   ├── build-docker/
│   │   └── SKILL.md
│   └── deploy-staging/
│       └── SKILL.md
├── src/
└── README.md
```

Update your Skill Loader to search:

```python
self.search_paths = [
    Path.cwd() / ".project-skills",  # Project-specific (highest priority)
    Path.cwd() / ".skills",          # Global via symlink
    Path.home() / ".agent-skills",
    Path("/Users/lloyd/ai/skills/skills"),
]
```

This way:
- Global skills are always available
- Projects can override or extend
- Team members get project skills automatically

## Debugging: Verify Symlinks Are Working

```bash
# Check if symlink exists and is valid
ls -la my-project/.skills

# Should show: .skills -> /Users/lloyd/ai/skills/skills

# Verify it points to the right place
readlink my-project/.skills

# Should output: /Users/lloyd/ai/skills/skills

# List what the agent sees
ls my-project/.skills/

# Should show: cs2-tradeup-buyer  docker-debug  github-pr-review
```

## Multi-Repo Strategies

### Strategy 1: Monorepo with Shared Skills

```
monorepo/
├── .skills -> ~/ai/skills/skills
├── services/
│   ├── service-a/
│   ├── service-b/
│   └── service-c/
```

All services inherit the global skills. Services can add `.project-skills/` if needed.

### Strategy 2: Multiple Skill Repos

If you want multiple skill repos (public, private, experimental):

```bash
# global symlink points to public
ln -s ~/ai/skills/skills .skills

# but Skill Loader also searches private
self.search_paths = [
    Path.cwd() / ".skills",
    Path.home() / ".agent-skills-private",
    Path("/Users/lloyd/ai/skills/skills"),
]
```

### Strategy 3: Team Skill Registry

For teams, point to a shared registry:

```bash
# All team members:
ln -s /mnt/shared/team-skills/.skills .skills

# Or use an MCP server:
self.search_paths = [
    Path.cwd() / ".skills",
    f"mcp://internal-skills-server/api/skills",
]
```

## Maintenance: Keeping Symlinks Fresh

### Weekly Sync Script

```bash
#!/bin/bash
# ~/bin/sync-skills

cd ~/ai/skills
git pull --rebase

# Update all project symlinks
find ~/projects -name ".skills" -type l | while read link; do
  project=$(dirname "$link")
  echo "✓ $project has skills available"
done
```

### CI/CD Integration

In your project's CI/CD:

```yaml
# .github/workflows/setup.yml
jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - name: Clone agent-skills repo
        run: git clone https://github.com/YourName/agent-skills ~/ai/skills

      - name: Create .skills symlink
        run: ln -s ~/ai/skills/skills ./.skills

      - name: Verify skills available
        run: ls .skills/
```

## FAQ

**Q: Should .skills be in .gitignore?**
A: Only if you're NOT committing the symlink. I recommend committing it, so don't ignore it.

**Q: Can symlinks cause issues on Windows?**
A: Use `mklink /D .skills C:\path\to\skills` instead of `ln -s`. Git handles both.

**Q: What if someone deletes the symlink by accident?**
A: It's just a symlink, easy to recreate: `ln -s ~/ai/skills/skills .skills`

**Q: Can we point to a GitHub repo instead?**
A: Yes! Clone the repo instead: `git clone https://github.com/LloydNicholson/agent-skills ~/ai/skills` then create symlinks.

**Q: Should projects share one .skills or each have their own?**
A: One global `.skills` pointed to by all projects (via symlinks). Simpler, faster, everyone synced.

---

## Summary

✅ **One global skills repo** at `~/ai/skills/`
✅ **All projects symlink** to `.skills -> ~/ai/skills/skills`
✅ **Projects can override** with `.project-skills/` if needed
✅ **Version skills** when breaking changes occur
✅ **Automate symlink creation** in onboarding / CI
✅ **Keep skills in Git** so teams stay synced
