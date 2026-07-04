---
name: skill-optimizer
description: Iteratively optimize an existing skill by running N parallel subagents against a test input, comparing artifacts across runs, identifying real issues (failures reproduced in ≥ N/3 of runs), and fixing the skill via a 16-step revision flow. Loops until M consecutive empty rounds = convergence. Use whenever the user asks to iterate, optimize, refine, harden, improve, validate, or empirically debug an existing skill against a real task — even if they only say "make this skill better" or "find bugs in this skill". Do NOT use for creating a new skill from scratch (use skill-creator instead).
---

# Skill Optimizer

This skill hammers an existing skill against a real task: spawn parallel runs, compare artifacts, fix the real issues, repeat. Every fix goes through the `finding-resolution` skill's 16-step flow.

**Output language:** match the user's conversation language for AskUserQuestion, narration, and the final report. Subagent prompts and this skill's body stay English.

---

## Pace

Echo the mantra at cue points; keep the pacing mindset everywhere else.

**Mantra:**

> Take a breath. Step by step, slowly, think each one through until it's clear. Stay calm, keep your established rhythm, neither hurried nor sluggish.

**Cue point** — parent restates mantra verbatim as first line, before each outer-loop Step 3 (Classify). (`finding-resolution` echoes its own copy of this mantra before each issue's inner Step 1 — that's its responsibility, not this skill's.)

Skipping the echo because "I just did one" is the failure mode.

---

## Inputs (3 required + 1 conditional)

1. **Target skill path** — absolute path to skill directory.
2. **Test input** (required only when target skill needs input) — file paths or inline task description; subagent prompt embeds verbatim. Mark N/A otherwise; subagent prompt omits the test-input line.
3. **N** — parallel subagents per round, ≥ 3 (below 3 the `≥ N/3` threshold collapses; at N=3 Suspected also collapses).
4. **M** — consecutive empty rounds to converge. Recommended ≥ 2.

---

## Phase 0: Preparation

Paths are cwd-relative — run from somewhere you can write to `docs/`.

1. **Sort out ambiguity** — ask one question at a time with `AskUserQuestion` until the inputs and intent are clear; check whether the target skill needs input (mark Test input N/A if not). Getting this wrong burns N×M runs.
2. **Backup** — once before iteration 1: if `docs/skill-optimizer-runs/<target-skill-name>/<YYYY-MM-DD>/skill-backup/` for today does not exist, run `mkdir -p docs/skill-optimizer-runs/<target-skill-name>/<YYYY-MM-DD>/skill-backup && cp -r <target-skill-path>/. docs/skill-optimizer-runs/<target-skill-name>/<YYYY-MM-DD>/skill-backup/` (the `/.` form includes dotfiles). Each run on a different day gets its own `<YYYY-MM-DD>/` workspace; same-day re-runs reuse it and skip the backup — never overwrite.

   "Latest date" elsewhere in this skill means the lexically maximal `<YYYY-MM-DD>/` subfolder under `<target-skill-name>/` (ISO date sort = chronological order). "Latest backup" means `<latest-date>/skill-backup/`.

---

## Outer Loop: Iterate Until Convergence

Start `empty_rounds = 0`, `iteration = 0`. Hard cap: break when `iteration > 20`.

```
loop:
  iteration += 1
  if iteration > 20: break (hard cap)
  step 1: spawn N subagents in parallel, each running target skill against test input
  step 2: cross-run artifact comparison
  step 3: classify issues (real if ≥ N/3 runs show the same observed failure)
  step 4: cp -r <target-skill-path>/. /tmp/skill-optimizer-snapshot-<target-skill-name>-<YYYY-MM-DD>-iter-<iteration>
          if real_issues non-empty: invoke `finding-resolution` skill (via `Skill` tool) with the whole real_issues batch, Step 2 context, and target-skill-path; it drains its own queue (including anything its Step 15/16 discovers) and reports back fixed vs. unfixed issues
          unfixed = finding-resolution's reported unfixed list (issue + reason: non-issue / crashed / declined)
          if `diff -r /tmp/skill-optimizer-snapshot-<target-skill-name>-<YYYY-MM-DD>-iter-<iteration> <target-skill-path>` shows any change: empty_rounds = 0
          else: empty_rounds += 1
          outcome_label = nothing-found | fixing | rejected-all | partial | errored
  step 5: write iteration summary.md
  if empty_rounds >= M: break (converged)
  continue loop

after loop: write <YYYY-MM-DD>/report.md and present handoff
```

### Step 1: Spawn N parallel subagents

Each subagent is dispatched via the `Agent` tool with `run_in_background: true`. Parent substitutes `<target-skill-path>`, `<test input verbatim>` (if N/A, omit the test-input line entirely instead of substituting), `<target-skill-name>` (basename of `<target-skill-path>`), `<YYYY-MM-DD>` (today's date), `<iteration>`, `<i>`. Skeleton:

```
Read the target skill SKILL.md at <target-skill-path>/SKILL.md.
Run the skill against this test input: <test input verbatim>
Save all artifacts to: docs/skill-optimizer-runs/<target-skill-name>/<YYYY-MM-DD>/iteration-<iteration>/run-<i>/
Follow the skill's instructions exactly. Do not skip phases. Do not declare done without writing artifacts.

When complete, your final text IS the return value. Report:
1. Final output path
2. Exit gates defined by skill, with pass/fail
3. Any phase not completed and why
4. Harness friction (tool blocked, dispatch failed, etc.)
```

Wait for all background task notifications (do not poll).

If a subagent fails to produce artifacts (no notification / crash / dispatch error / wrote nothing to its run directory), try one replacement. If that also fails, drop it. Let K = total drops this round. If (N-K) ≥ 3 carry on. If (N-K) < 3 abort (see Stopping Conditions).

### Step 2: Cross-run artifact comparison

`Read` artifacts across run directories to compare findings. Also parse each subagent's return text for self-reports. Stay observation-driven.

Write `observations.md` (language follows conversation; bold elements stay English):

**Per-run** (table):
- Run ID
- Artifacts produced
- Exit gates pass/fail (N/A if none)
- Self-reported deviations
- Harness friction
- Excluded runs (reason)

**Cross-run** (free prose feeding Step 3):
- Shared findings (which N)
- Severity divergence
- Finding count + scope spread
- Recurring issues from prior summary.md (skip on iteration 1)
- Candidate issues for classification

### Step 3: Classify issues

Echo the Pace mantra verbatim (see `## Pace`) before continuing.

For each candidate issue (from observations.md Cross-run):

- **Real issue**: ≥ N/3 runs with artifact evidence — fix.
- **Suspected**: < N/3 runs — defer. At N=3 this never fires.
- **Hypothetical (file-imagined)**: subagent's text-only claim, no artifact backing — drop.

Examples (N=3, threshold ≥ 1; any artifact-backed hit counts):
- 1/3 runs skip same instruction with artifact evidence → real
- 1/3 self-report tool blocked, no artifact → hypothetical (drop)
- "method count 7 vs 6" from reading skill, no failure → hypothetical

### Step 4: If real issues, fix; else count empty round

If `real_issues` is non-empty, invoke the `finding-resolution` skill (via the `Skill` tool) once with the whole batch: pass the full `real_issues` list as Issues, the target skill's stated purpose (frontmatter description + intro paragraph) as Context, and `<target-skill-path>` as Target location. It drains its own queue one issue at a time through its 16-step flow — including any related issues its own Step 15/16 discovers — and reports back which issues were fixed (file:line + summary) versus unfixed (issue + reason: `non-issue` / `crashed` / `declined`).

`empty_rounds` is updated on Step 4 **outcome**, measured by `diff -r` of the target skill directory against a `cp -r` snapshot taken at Step 4 entry (same snapshot mechanism as Phase 0 backup). If any file differs, `empty_rounds = 0`; otherwise `empty_rounds += 1`. This automatically covers:
- All issues resolve as `non-issue` → no edit → counted as empty
- Crash / declined / Edit silent-failed → no edit → counted as empty
- An edit lands then reverts in the same iter → net zero → counted as empty
- Partial drain (1 lands, 4 unfixed) → non-empty → not empty
- Reference-file-only edit → still counts (whole dir snapshotted)

(Whitespace-only / comment-only edits are not normalized out — skill optimizations land token-meaningful changes by design; pure formatting drift is treated as progress.)

Outcome label per iter (written to summary.md):
- `nothing-found`: real_issues was empty at iter start
- `fixing`: ≥1 edit landed
- `rejected-all`: real_issues non-empty, all unfixed
- `partial`: real_issues non-empty, some applied + some unfixed
- `errored`: `finding-resolution` reports every issue crashed / declined

### Step 5: Write iteration summary.md

Write `docs/skill-optimizer-runs/<target-skill-name>/<YYYY-MM-DD>/iteration-<iteration>/summary.md`:

- **Iteration**: <iteration>
- **Runs**: succeeded count / excluded (with reason)
- **Real issues** found: title + ≥ N/3 evidence
- **Suspected issues** deferred: title + source + reason
- **Changes applied**: file:line + 1-line diff per change
- **Unfixed**: issue title + reason (`non-issue` / `crashed` / `declined`)
- **Outcome label**: `nothing-found` / `fixing` / `rejected-all` / `partial` / `errored`
- **Diff stat**: file count / line count (from `diff -r` against Step 4 snapshot)
- **empty_rounds** counter

Language follows conversation; bold elements stay English.

---

## Inner Flow

Every real issue's fix goes through the `finding-resolution` skill's 16-step flow (Verify facts → Context → Problem → Classify → Research → Intervention type + initial proposal → Expand check → Expanded version → Trim check → Trimmed version → KISS check → First-principles version → Apply → Verify → Read-through → Regression check). That skill owns the step definitions, the pendulum discipline, dispatch modes, and the 5 KISS rules — see outer Step 4 for how this skill invokes it.

---

## Workspace Structure

```
docs/skill-optimizer-runs/<target-skill-name>/
└── <YYYY-MM-DD>/                  # one workspace per day; same-day re-runs reuse
    ├── skill-backup/              # cp -r of target skill before iteration 1
    ├── iteration-1/
    │   ├── observations.md
    │   ├── run-1/
    │   ├── run-2/
    │   ├── ...
    │   └── summary.md
    ├── iteration-2/
    ├── ...
    └── report.md
```

---

## Final Report Schema (`report.md`)

Write `docs/skill-optimizer-runs/<target-skill-name>/<YYYY-MM-DD>/report.md` in conversation language; bold elements stay English:

- **Summary** — target skill name, N, M, total iterations, total real issues fixed, exit reason (Converged at iteration X / Hard cap / ABORTED at iteration X).
- **Convergence reason**: `edit-stable` (M empty rounds with edits earlier) / `dry` (M empty rounds, zero edit ever) / `cap` (iter > 20).
- **Per-iteration log** — concatenate each `iteration-<iteration>/summary.md` in order.
- **Before/after stats** — target skill line + file count vs `<latest-date>/skill-backup/`.
- **Rollback hint** — `cp -r docs/skill-optimizer-runs/<target-skill-name>/<latest-date>/skill-backup/. <target-skill-path>/`.

---

## Stopping Conditions

- **Converged** (`empty_rounds >= M`): write final report.md; present handoff.
- **Hard cap reached** (`iteration > 20`): write report.md noting non-convergence; surface to user.
- **Aborted** (surviving runs < 3): partial report.md already written; surface to user.

Once stopped, drop into conversation: report path, total iterations, total issues fixed, exit reason. Wait silently. Don't insert `AskUserQuestion` at handoff.
