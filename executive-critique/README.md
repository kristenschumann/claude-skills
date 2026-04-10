# /executive-critique

Pressure-test any plan or idea through two executive lenses: operational (execution, timelines, dependencies) and strategic (ROI, ambition, customer psychology). Surfaces tension between perspectives.

## Install

Copy the SKILL.md file into `~/.claude/skills/executive-critique/SKILL.md` and it'll work in your next Claude Code session.

Or if you have the skills CLI:

```bash
npx skills add kristenschumann/claude-skills --skill executive-critique -a claude-code -y
```

## Usage

```
/executive-critique operational [your plan]
/executive-critique strategic [your plan]
/executive-critique both [your plan]
```

- **operational** — execution, timelines, dependencies, cross-functional coordination
- **strategic** — ROI, ambition, customer psychology, competitive positioning
- **both** — both perspectives, plus where they'd disagree
