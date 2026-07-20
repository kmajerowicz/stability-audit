# Stability Audit — Report Template

Copy this skeleton into the output report and fill it in. Delete these top instructions and any
sections that genuinely don't apply (note why if you drop one). Keep the ordering: the triage
table and "why it's flaky" go near the top because they're what stakeholders read first.

Conventions used throughout:
- Reference code as `file:line`.
- Severity: **C**ritical / **H**igh / **M**edium / **L**ow.
- Effort: **S** (≤½ day) / **M** (1–3 days) / **L** (multi-day).
- Confidence: **Confirmed** (visible in code) / **Inferred** (depends on external behavior — see verification plan).
- "Ship-blocker?" = would you withhold a launch over it.

---

# <service> — Engineering Bug Report

**Scope:** `<dir>/` (list the subsystems audited) plus any shared package it depends on.
**Audience:** <who this is for, and the deadline context if any>.
**Reviewed against:** commit `<sha>` (`<commit date>`). **Report date:** `<date>`. Findings reference `file:line` at this commit — re-anchor if the code has moved.
**TL;DR:** <one paragraph naming the single structural root cause, the symptom class it produces (incl. any "sometimes it works" flakiness), and how much of it is one fix vs many>.

---

## Triage summary

> One-screen index into the detail below. Additional to, not a replacement for, the per-finding write-ups.

| ID | Title | Sev | User-facing impact | Effort | Confidence | Ship-blocker? |
|---|---|---|---|---|---|---|
| #1 | … | C | … | L | Confirmed | Yes |
| … | … | … | … | … | … | … |
| U1 | <user-reported> | H | … | S | Confirmed | Yes |

**If you cut scope to the deadline:** <name the ship-blockers and what survives a scope cut; point to the remediation section>.

---

## 0. Why it's flaky ("sometimes it works, sometimes it doesn't")

> Include this section only if the system has intermittent failures. Reduce them to deterministic
> causes with probabilistic outcomes. Typical buckets: (1) LLM re-transcription of structured
> data; (2) order-dependent logic over unordered data; (3) caches that outlive what they cache.
> Map each user-visible flaky symptom to its bucket.

---

## 1. Critical
## 2. Security
## 3. High
## 4. Medium
## 5. Low / cleanup

> For each finding use this shape:
>
> ### #N — <short title>
> **Where:** `file:line` (the relevant spots).
> **Mechanism.** How it actually happens, in causal terms.
> **Why intermittent.** (Only if it is.) What makes the same input sometimes pass, sometimes fail.
> **Impact.** What the user/operator sees; how bad.
> **Fix.** The *stable* fix. If a hotfix is tempting, name it and say why you're recommending against it.
>
> Low/cleanup items can be one-liners with `file:line`.

---

## 6. User-reported bugs (verified against the code)

> Only if the user supplied a list. For each:
>
> ### U<N> — <reporter's framing of the symptom>
> - **Status:** Confirmed / Inferred. New finding, or "same root cause as #X".
> - **Symptom (reporter):** their words.
> - **Root cause.** The mechanism in code, `file:line`. If the symptom proves a root cause (e.g. a
>   divergence between two outputs), say so explicitly.
> - **Why intermittent.** If applicable.
> - **Stable solution.** Cause → problem → fix. Cross-reference the structural finding it belongs to.

---

## 7. Verification & research plan (do this before coding the Inferred fixes)

### 7.1 Verified now (no further research needed)
| Fact | Source (URL / issue) | Unblocks |
|---|---|---|
| … | … | … |

### 7.2 Must verify against the live system / external API
> The cheapest source is usually existing logs — point at them first. Then a sandbox/repro.

| Open question | Why it matters | How to verify | Design fork it decides |
|---|---|---|---|
| … | … | logs: `<pattern>` / sandbox: `<test>` | option A vs B |

### 7.3 Recommended process
1. Harvest logs (no env needed) — resolves what it can from real data.
2. Sandbox/repro for the rest. **Confirm a sandbox exists**, or note testing is against production.
3. Write each Inferred fix spec-first: its verified premise stated in the PR description.
4. Land the Confirmed/independent fixes in parallel now.

---

## 8. Recommended remediation order

> Lead with the structural refactor that dissolves a whole class of findings, then independent
> additive fixes, then security must-dos, then the rest. Number them; note dependencies.

1. **<structural refactor>** — fixes #…, and user-reported U…. <one line on the core change>.
2. **<additive fix>** — independent, low-risk, ship early.
3. **<security must-do>** — isolated, ship immediately.
4. … 

**Dependency note:** <which findings fall out of one refactor vs are independent>.

> **Scope honesty — state what the big refactor does NOT cover.** Call out findings that live on a
> different code path and must be scoped as their own items, so nobody assumes the refactor absorbs
> them.
