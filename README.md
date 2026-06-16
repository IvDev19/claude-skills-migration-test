# claude-skills-migration-test

Test repo for validating the `.claude/skills/` symlink migration approach.

## What this tests

The Claude Code docs confirm that `.claude/skills/` within an `--add-dir` directory
is loaded automatically, making skills appear in the `/` menu with no namespace prefix.
This repo validates that claim and answers the `${CLAUDE_SKILL_DIR}` resolution question.

## Structure

```
skills/
  skill-creator-per/SKILL.md    ← source of truth (verbatim copy + debug comment)
  skill-promote/SKILL.md        ← source of truth (verbatim copy + debug comment)
.claude/
  skills/
    skill-creator-per -> ../../skills/skill-creator-per  (symlink)
    skill-promote     -> ../../skills/skill-promote      (symlink)
```

## Verification checklist

### 1. Symlinks committed correctly
```bash
git ls-tree HEAD .claude/skills/
# Expected: mode 120000 (symlink) for each entry
```

### 2. ${CLAUDE_SKILL_DIR} resolution
Connect as `--add-dir`, invoke each skill manually:
```
/skill-creator-per
/skill-promote
```
Each SKILL.md contains:
```
<!-- debug: skill dir = ${CLAUDE_SKILL_DIR} -->
```
- If `${CLAUDE_SKILL_DIR}` = `.claude/skills/skill-creator-per/` → symlink path used
- If `${CLAUDE_SKILL_DIR}` = `skills/skill-creator-per/` → canonical path used

Either is fine for this migration since no skill uses `${CLAUDE_SKILL_DIR}` today,
but the result determines whether future skills with bundled resources would need
path-awareness.

### 3. Native / menu discovery
```
/add-dir /path/to/claude-skills-migration-test
```
Type `/sk` — confirm `/skill-creator-per` and `/skill-promote` appear without
a `pluginname:` prefix.

### 4. disable-model-invocation
Both skills have `disable-model-invocation: true`. Ask Claude to create a new skill
or promote a skill without using the `/` command. Confirm neither fires automatically.
