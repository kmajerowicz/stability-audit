---
name: stability-audit
description: >
  Deep architectural audit of a codebase or service that produces a root-cause engineering
  bug report — not a hotfix list. Use when asked to "audit", "review the architecture",
  "find the bugs", "figure out why it's flaky / unstable / sometimes works", investigate a
  class of failures, or produce a stability/bug report for a service or module. Reads the
  code file-by-file, traces data flow across boundaries to separate root causes from
  symptoms, verifies library/API assumptions against primary sources, distinguishes
  Confirmed (in-code) from Inferred (depends on external behavior) findings, proposes a
  stable fix for each, and writes a structured report with a triage table and a
  verification + remediation plan. At the start it asks the user for repo context, known
  bugs, runtime evidence (screenshots/logs/transcripts), and "worked before" context, since
  the highest-value findings often come from runtime symptoms not visible in static code.
  Originally developed on a multi-variant AI agent system, works on any codebase.
---

# Stability Audit

Produce a **root-cause engineering bug report** for a target codebase/service. The deliverable
is a durable markdown report (see `REPORT_TEMPLATE.md` in this skill folder). The goal is a
report a dev team can execute against under deadline **without coming back with questions** —
each finding has the mechanism, `file:line`, why it's intermittent, the impact, and a *stable*
fix (never a hotfix).

## Inputs

- **Target** (required): the directory/service to audit (e.g. `src/`, `services/api/`).
- **Repo / tech-stack context** (high-value): what the service does and its stack, especially if
  not obvious from the docs. A 30-second answer prevents the audit from walking in with wrong priors.
- **User-reported bugs / symptoms** (high-value): symptoms users have hit. Verify each against
  the code; don't take the reporter's *diagnosis* at face value (the agent/user may misattribute).
- **Runtime evidence** (high-value): screenshots, chat transcripts, logs, error messages. Some
  findings are **invisible in static code** and only show up here (see principle 8).
- **Recent-change context**: anything like "it worked a few days ago" — a strong signal of
  nondeterminism (regenerated output) rather than static breakage. Ask for it.
- **Usage/demo context**: how it's used, what must work reliably, any upcoming demo. Changes
  which findings matter most.
- **Output path** (default `docs/<target>-bug-report.md`).

**Don't wait for these to be volunteered — ask.** At the start of a run (Phase 0), proactively
request repo context, the bug list, and any runtime evidence. Most of the highest-value findings
in past runs came from user-supplied symptoms and screenshots, not from reading code alone. If the
user has none, proceed with the static audit and say so — but always offer the slot.

## Operating principles (read first — these are what make the report good)

1. **Root cause, not symptom.** Most "lists of bugs" are one or two diseases wearing many
   costumes. Before writing findings, look for the *single structural decision* that spawns
   multiple symptoms. In past audits, ~7 symptoms reduced to one cause (an LLM re-authoring
   typed data at write boundaries). Find that spine first; it changes everything downstream.
2. **"Sometimes it works" = find the nondeterminism source.** Intermittent failures are not
   "infra flakiness." They almost always reduce to deterministic causes with probabilistic
   outcomes. The usual three: (a) an LLM re-transcribing structured data it should read
   verbatim; (b) order-dependent logic over something not actually ordered; (c) a cache that
   outlives the thing it caches. Name the cause, not the symptom.
3. **No hotfixes.** For every finding give the *stable* fix — usually "move the guarantee from
   the model/prompt into code/state/types." If a tempting hotfix exists, name it and say why
   you're recommending against it. Stability = remove the failure mode, don't retry around it.
4. **Confirmed vs Inferred.** Label every finding. **Confirmed** = visible in the code you read.
   **Inferred** = depends on external behavior (a third-party API, the rendered UI, a service's
   responses) you could not observe. Never propose coding an Inferred fix without a verification
   step first — that's how a plausible fix ships wrong.
5. **Verify library/API assumptions against primary sources.** Do NOT trust your memory of a
   framework's behavior, especially version-specific. In past audits, the "obvious" retry fix
   would have silently no-op'd because a library ignores tool-error config when middleware is
   present — found only by reading the GitHub issue. Check the pinned version, then the docs.
6. **The cheapest ground truth is often already logged.** Before proposing a sandbox/repro,
   check whether the service already logs the thing you need (schemas, raw API responses).
   Point the team at harvesting existing logs first.
7. **Use `file:line` everywhere** and **call `advisor`** before committing to the root-cause
   interpretation and again before declaring the report done.
8. **Some findings live only at runtime — ask for evidence.** Static reading cannot see
   nondeterministic generation, misattribution, or "worked days ago" regressions. In one past
   audit, the headline runtime bug (a spatial-SQL join the agent re-authored each run and then
   *blamed on the data*) was invisible in the code and surfaced only from user screenshots. Always
   solicit symptoms + logs/transcripts, and treat the system's own self-diagnosis as a hypothesis
   to verify, not a fact (it may claim to "check logs" it can't read, or blame data for its own bug).
9. **For security findings, separate the *mode* from the *scope*.** "Read-only" ≠ "scope-limited";
   "authenticated" ≠ "authorized." Verify write-safety AND read-scope AND per-user authz as
   separate questions. In one audit, SQL was verified write-safe server-side, but the *read scope*
   was "anything the service account's IAM allows" with no per-user authz — the actual headline
   risk. The boundary that matters is the one enforced by IAM/code, not by a prompt rule.

## Process

### Phase 0 — Intake (ask before reading)
Before diving into code, ask the user for the inputs that produce the best findings — use
`AskUserQuestion` or a short direct prompt:
- **Repo / tech-stack context.** What does this service do? What's the stack (framework, DB,
  external APIs, LLM if any)? A brief answer prevents the audit from walking in with wrong priors.
  If the README/CLAUDE.md covers it, skip this question and say you'll learn it in Phase 1.
- **Known bugs / symptoms?** Anything users have hit, in their own words.
- **Runtime evidence?** Screenshots, transcripts, logs, error text.
- **"Worked before"?** Anything that recently regressed — flags nondeterminism.
- **Usage / demo context?** What must be reliable; any deadline or demo that reprioritizes severity.

If the user supplies bugs/evidence, fold them into Phase 4 and weight the audit toward them. If
they have none, say "proceeding with a static audit; runtime evidence would likely surface more"
and continue. Never block on intake — but never skip offering it.

### Phase 1 — Orient (don't skip; cheap)
- Read the target's own `CLAUDE.md`/`README` and the repo-root `CLAUDE.md`. Understand what the
  service is *supposed* to do and its claimed architecture.
- List the source files. Note the entry point, the agent/orchestrator, tool definitions, state/
  schema, prompts, external clients, config, and any shared package it depends on.
- Get the commit SHA (`git rev-parse --short HEAD`) and date — they go in the report header.

### Phase 2 — Systematic read (every file, prompts included)
Read the whole surface, not a sample. Order that works: entry point → turn/dispatch logic →
orchestrator/agent → subagents/tools → batch/write tools → state & typed schemas → **all prompt
files** (base and per-deployment overrides) → cards/UI builders → external clients → config →
shared package. Prompts are code here — read them; the language/tone/guardrail bugs live there.

### Phase 3 — Trace data flow & find the spine
- Follow the important data end-to-end, paying special attention to **boundaries**: where typed
  data becomes free text, where an LLM re-emits something it was given, where state is read vs
  re-derived, where one component's output is re-typed by another.
- At each boundary ask: *is the guarantee enforced by code/types, or is it hoping the model
  reproduces it?* The latter is where flakiness and data-loss live.
- Group symptoms under structural causes. Write the one-sentence root cause (the "TL;DR" and the
  "Why it's flaky" section come from here).

### Phase 4 — Verify user-reported bugs & runtime evidence (if any supplied)
For each reported symptom: find the mechanism in code, cite `file:line`, and state whether it's
a **new** finding or a **symptom of an existing root cause**. The divergence between "what works
in one place vs fails in another" is often the proof of the root cause (e.g. text correct in the
UI but wrong in the external system ⇒ proof the write path re-authors it). Mark Confirmed/Inferred.
- **Distrust the reported diagnosis; verify the mechanism.** Users — and the system itself —
  often mis-attribute (e.g. the agent blamed "the data" for a join it wrote wrong). Re-derive the
  cause from code + evidence; correct the diagnosis in the report if it's wrong, and say so.
- **Read transcripts/screenshots for fingerprints.** "All X failed, not some" ⇒ systematic
  transform error, not partial data corruption. "Worked days ago" ⇒ regenerated output, not static
  breakage. A simple query that worked at T1 and fails at T2 ⇒ sticky degradation in a shared layer.
- **Point at the real logs, not the system's self-report.** If the service logs tool inputs/
  outputs, the exact generated query/payload is ground truth; harvest it (you may not have access —
  give the user the exact command).

### Phase 5 — Research the unknowns
- Identify every assumption the fix depends on that you can't see in code: library/framework
  behavior, external API contracts, UI rendering, ordering/expiry guarantees.
- Verify what's publicly verifiable: check the **pinned dependency version**, then official docs,
  then GitHub issues for version-specific gotchas. Use primary sources; cite URLs.
- For what's *not* publicly verifiable (the service's own external integration), write a
  **verification plan**: what to check, *how* (logs first, then a sandbox/repro), and which design
  decision each answer unblocks (the "decision fork"). Flag whether a sandbox exists.
- Surface any *new risk* the research reveals (e.g. an API requiring a normalize/upload step the
  code skips) as an explicit verification item, not a silent assumption.

### Phase 6 — Classify & propose stable solutions
For each finding: **Where** (`file:line`) · **Mechanism** · **Why intermittent** (if it is) ·
**Impact** · **Stable fix**. Severity C/H/M/L. Confidence Confirmed/Inferred. Effort S/M/L.
Note dependencies between fixes (which ones fall out of one refactor vs are independent).

### Phase 7 — Triage & sequence
- Build the triage table (top of report): ID · title · severity · user-facing impact · effort ·
  confidence · ship-blocker?
- Write the remediation order: lead with the structural refactor that dissolves a whole class,
  then independent additive fixes, then security must-dos, then the rest.
- **Explicitly state what the big refactor does NOT cover** — scope-overstatement is a real risk
  (e.g. a write-path refactor doesn't fix generation-time/edit bugs). Call out items that must be
  scoped separately.

### Phase 8 — Write the report
Use `REPORT_TEMPLATE.md`. Write to the output path. It's the deliverable — make it durable before
calling `advisor` for the final review.

## Anti-patterns (don't do these)
- Listing symptoms as independent bugs without finding the shared cause.
- Proposing retries/guards around a nondeterministic step instead of removing the nondeterminism.
- Recommending a fix whose correctness depends on unverified external behavior, with no
  verification step.
- Trusting your memory of a library's API instead of checking the pinned version + docs.
- Re-reading a sample of files and extrapolating. Read the whole surface, prompts included.
- Implying one refactor fixes everything when some bugs live on a different path.
- Starting the audit without asking for known bugs / runtime evidence first.
- Repeating the system's self-diagnosis as fact instead of verifying the mechanism.
- Treating "read-only" / "authenticated" as if they meant "scope-limited" / "authorized."

## Worked examples
Past audits from a multi-variant AI agent system (shared infra: agent framework, vector DB,
external APIs, Postgres checkpointing):

- **Complex variant** — a single structural spine (~7 symptoms, one root cause: LLM re-authoring
  typed data at write boundaries). Reference for: triage table format, "why it's flaky" section,
  Confirmed/Inferred labels, verification plan, and scope-honesty notes.
- **Simpler variant** — no single spine; multiple independent findings. Reference for: an
  access-scope security finding (mode vs scope), a fail-open finding, and a runtime-only finding
  surfaced from user screenshots (invisible in static code).

## Reuse across services that share infrastructure
When auditing multiple services that share a common package or infra layer, shared-package
findings often transfer — but verify before assuming. Even "identical" shared code can be pinned
to different versions across services, or one service may wrap it differently. Re-read each
service's orchestration, state, and prompts from scratch. Only carry over shared findings after
confirming the shared code is actually the same at the revision you're auditing.
