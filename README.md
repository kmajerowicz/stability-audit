# stability-audit

A Claude Code skill that produces a root-cause engineering bug report for a codebase or
service — not a hotfix list. It reads the target file-by-file, traces data flow across
boundaries to separate root causes from symptoms, verifies library/API assumptions against
primary sources, distinguishes Confirmed (in-code) from Inferred (depends on external
behavior) findings, and writes a structured report with a triage table and a verification +
remediation plan.

Originally developed on a multi-variant AI agent system; works on any codebase.

## Install in Claude Code

Prerequisites: [Claude Code](https://code.claude.com/docs/en/overview) and Git installed.
Run the installation commands below in your terminal. They download this skill;
installing or updating an AI coding CLI does not install the skill.

Choose **one** installation scope. Both options include `SKILL.md` and the supporting
`REPORT_TEMPLATE.md` in the directory Claude Code uses to discover the skill.

### Global (available in every project)

```bash
mkdir -p "$HOME/.claude/skills"
git clone https://github.com/kmajerowicz/stability-audit.git \
  "$HOME/.claude/skills/stability-audit"
```

### Project-scoped (available only in one project)

Run this from the root of the project you want to audit:

```bash
mkdir -p .claude/skills
git clone https://github.com/kmajerowicz/stability-audit.git \
  .claude/skills/stability-audit
```

These commands are for a fresh installation. If the destination already contains a
clone of this repository, use the update command below. If you previously copied the
skill there, move that copy elsewhere before cloning.

### Update

For a global installation:

```bash
git -C "$HOME/.claude/skills/stability-audit" pull --ff-only
```

For a project installation, run from the project root:

```bash
git -C .claude/skills/stability-audit pull --ff-only
```

Updates are manual: cloning or symlinking the repository does not download future
changes automatically.

### Use

Start a new Claude Code session in the project you want to audit. In the Claude Code
prompt, type:

```text
/stability-audit
```

You can also ask Claude to "audit this service" or "find out why it's flaky."
If the command is missing, check that `SKILL.md` is directly inside
`~/.claude/skills/stability-audit/` (global) or `.claude/skills/stability-audit/`
(project-scoped), as described in the [Claude Code skills documentation](https://code.claude.com/docs/en/skills).

## What's in here

- `SKILL.md` — the skill definition: operating principles, the 8-phase process, and
  anti-patterns.
- `REPORT_TEMPLATE.md` — the report structure the skill fills in (triage table, findings,
  verification plan).
