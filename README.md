# stability-audit

A Claude Code skill that produces a root-cause engineering bug report for a codebase or
service — not a hotfix list. It reads the target file-by-file, traces data flow across
boundaries to separate root causes from symptoms, verifies library/API assumptions against
primary sources, distinguishes Confirmed (in-code) from Inferred (depends on external
behavior) findings, and writes a structured report with a triage table and a verification +
remediation plan.

Originally developed on a multi-variant AI agent system; works on any codebase.

## Install

Skills live in `.claude/skills/<name>/` inside your project, or in `~/.claude/skills/<name>/`
to make them available globally across all projects.

**Project-scoped:**
```bash
mkdir -p .claude/skills
cp -r path/to/stability-audit .claude/skills/
```

**Global (available in every project):**
```bash
mkdir -p ~/.claude/skills
cp -r path/to/stability-audit ~/.claude/skills/
```

**To stay current with updates**, clone this repo and symlink instead of copying:
```bash
mkdir -p ~/.claude/skills
ln -s /path/to/stability-audit ~/.claude/skills/stability-audit
```

Once the folder is in place, restart Claude Code. Invoke it with `/stability-audit` in the
prompt, or just ask Claude to "audit this service" / "find out why it's flaky."

## What's in here

- `SKILL.md` — the skill definition: operating principles, the 8-phase process, and
  anti-patterns.
- `REPORT_TEMPLATE.md` — the report structure the skill fills in (triage table, findings,
  verification plan).
