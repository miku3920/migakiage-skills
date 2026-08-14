---
name: skill-optimizer
description: Iteratively optimize an existing skill by running N parallel subagents against a test input, having six dimension-based reviewer groups compare artifacts across all runs, adversarially verifying each candidate issue, and fixing confirmed issues via a 20-step revision flow. Loops until M consecutive empty rounds = convergence. Use whenever the user asks to iterate, optimize, refine, harden, improve, validate, or empirically debug an existing skill against a real task — even if they only say "make this skill better" or "find bugs in this skill". Do NOT use for creating a new skill from scratch (use skill-creator instead).
---

# Skill Optimizer

Run a target skill against a real task N times in parallel, have six persistent reviewer subagents critique the runs and the skill file itself, adversarially verify what they find, fix confirmed issues via `finding-resolution`, and repeat until M rounds pass with nothing left to fix.

**Output language:** match the user's conversation language for AskUserQuestion, narration, and the final report. Subagent prompts and this skill's body stay English.

---

## Pace

Echo the mantra at cue points; keep the pacing mindset everywhere else.

**Mantra:**

> Take a breath. Step by step, slowly, think each one through until it's clear. Stay calm, keep your established rhythm, neither hurried nor sluggish.

**Cue point** — restate verbatim as the first line before each outer-loop Step 3. (`finding-resolution` echoes its own copy before its own **Explore** step; not this skill's concern.)

Skipping the echo because "I just did one" is the failure mode.

---

## Inputs (3 required + 1 conditional)

1. **Target skill path** — absolute path to skill directory.
2. **Test input** (required only when target skill needs input) — file paths or inline task description; subagent prompt embeds verbatim. Mark N/A otherwise; subagent prompt omits the test-input line.
3. **N** — parallel runner subagents per round, ≥ 3, so a reviewer has more than one run to tell a real, reproducing problem from one-off noise.
4. **M** — consecutive empty rounds to converge. Recommended ≥ 2.

---

## Phase 0: Preparation

Paths are cwd-relative — run from somewhere you can write to `docs/`.

1. **Sort out ambiguity** — ask one question at a time with `AskUserQuestion` until the inputs and intent are clear, including whether the target skill needs input (mark Test input N/A if not). Getting this wrong burns N×M runs.
2. **Backup**, once, before iteration 1: `mkdir -p docs/skill-optimizer-runs/<target-skill-name>/<YYYY-MM-DD>/skill-backup && cp -r <target-skill-path>/. docs/skill-optimizer-runs/<target-skill-name>/<YYYY-MM-DD>/skill-backup/` (the `/.` includes dotfiles) if that path doesn't already exist for today. A same-day re-run reuses the existing workspace and skips this.

   "Latest date" means the lexically maximal `<YYYY-MM-DD>/` under `<target-skill-name>/`; "latest backup" is that date's `skill-backup/`.

---

## Outer Loop: Iterate Until Convergence

Start `empty_rounds = 0`, `iteration = 0`. Hard cap: break when `iteration > 20`.

```
loop:
  iteration += 1
  if iteration > 20: break (hard cap)
  step 1: spawn N runner subagents (see Step 1)
  step 2: reviewer groups build the candidate list (see Step 2)
  step 3: verify candidates → real_issues (see Step 3)
  step 4: fix real_issues via finding-resolution; empty_rounds = 0 if target skill changed, else += 1 (see Step 4)
  step 5: write iteration summary.md (see Step 5)
  if empty_rounds >= M: break (converged)
  continue loop

after loop: write <YYYY-MM-DD>/report.md and present handoff
```

### Step 1: Spawn N parallel subagents

Dispatch each subagent via `Agent`, in parallel, substituting `<target-skill-path>`, `<test input verbatim>` (omit the line entirely if N/A), `<target-skill-name>` (basename of `<target-skill-path>`), `<YYYY-MM-DD>`, `<iteration>`, `<i>`:

```
Read the target skill SKILL.md at <target-skill-path>/SKILL.md.
Run the skill against this test input: <test input verbatim>
Save all artifacts to: docs/skill-optimizer-runs/<target-skill-name>/<YYYY-MM-DD>/iteration-<iteration>/run-<i>/
Follow the skill's instructions exactly. Do not skip phases. Do not declare done without writing artifacts.

When complete, your final text IS the return value — report any harness friction (tool blocked, dispatch failed, etc.).
```

Don't poll or sleep-wait on anything dispatched in parallel in this flow — the harness delivers the notification on its own.

A subagent producing no artifacts (no notification / crash / dispatch error / empty run directory) gets one replacement attempt, then gets dropped. K = drops this round; (N-K) ≥ 3 continues, (N-K) < 3 aborts (Stopping Conditions).

### Step 2: Reviewer groups

Six fixed subagents, one per dimension below — not scaled to N, never seeing each other's candidates or process. Each is dispatched via `Agent`, in parallel, exactly once, ever, on the first iteration reaching this step, then resumed every iteration after via `SendMessage` with that round's fresh runs — so it remembers what it already found without the parent restating it. Starts once every Step 1 runner is done.

Each reviewer reads, every iteration: all N `run-<i>/` directories in full, the Input #2 test input, and the target skill's own `SKILL.md` — critiqued as a document in its own right, not merely re-read the way Step 1's execution pass reads it.

1. **Correctness & flow group** — phase/step logic gaps; runs that deviated from `SKILL.md`'s defined path; every exit gate it defines, checked against actual run artifacts; whether each instruction's specificity matches its step's fragility (Anthropic's "degrees of freedom" — tight and prescriptive for a fragile step, latitude for a flexible one).
2. **Consistency group** — behavioral consistency across this round's N runs; `SKILL.md`'s own internal consistency (no contradictions, no dangling references); no information in it that will silently go stale.
3. **Mission-fit group** — mission-fit gaps, like every group finds problems, but the only one of the six that also proposes improvements where nothing's broken; also owns whether the frontmatter `description` is precise, third-person, and trigger-word-bearing enough to get the skill discovered at all — a precondition for serving the mission, so it belongs here rather than with the flow-logic checks in group 1.
4. **Robustness & exceptions group** — within the same passage of prose already written into `SKILL.md`, checked against four principles:
   - **Decide upfront** (no shoot-first-draw-target, no post-hoc remediation) — a process states what to do and what not to do before execution begins, never letting the behavior happen first and remediating a violation afterward; verification performed after a program or document is complete, used to confirm the artifact meets requirements already established beforehand, doesn't count as post-hoc remediation
   - **Binary decision** (no exception backdoors) — a rule's requirement on behavior is explicitly prohibition or permission, never a self-granted exception, proviso, or conditional branch; any exception must be explicitly specified by the user
   - **Necessary text only** (no examples, no rephrasing) — a rule keeps only the text genuinely necessary to execute it, with no example, explanation, or rewording carrying the same normative effect as a rule already there
   - **Direct replacement** (no reverse emphasis) — when a passage is revised or corrected, the superseded prior wording is never kept as a negation, contrast, or emphasis; the new version must directly replace the old one

   Only what's written, never what's missing.
5. **Architecture simplification group** — redundant, duplicate, splittable, or mergeable steps; unneeded complexity; `SKILL.md` carrying too many responsibilities; cross-references nested more than one level; a reference file long enough to need a table of contents but missing one.
6. **Security group** — sensitive data leaking through examples or instructions; instructions that could push the executing agent toward unsafe behavior.

For every candidate: assume it's real, then go find the evidence — don't just report what's already obvious. Zero, one, or many per group; say so if a group finds nothing, rather than inventing a candidate. Record each in full:

```
**<group name> - Candidate <n>**
- Where checked: <specific files, run-<i>/ paths, line numbers, sections searched>
- Observed fact: <what was actually seen — a fact, not a judgment>
- Assumed problem: <state it concretely, tied to this SKILL.md / these runs — not a restatement of the group's mandate>
- Verification result: <the evidence you went looking for, supporting or undercutting it>
- Judgment: <problem exists / improvement opportunity exists / no issue, with the reasoning>
```

Write `observations.md` (bold elements and the 5-field labels stay as given): a section per group, in the order above, each with its full candidate list or "no candidates this round" — plus **Harness friction** for any tool-blocked / dispatch-failed reports a reviewer surfaced.

Parent doesn't spot-check or filter here — that's Step 3's job entirely.

### Step 3: Verify issues

Echo the Pace mantra verbatim (see `## Pace`) before continuing.

If two or more groups independently raised the same underlying issue, merge them into one candidate before any verifier sees it — never verify the same real problem twice. Parent picks which group's verifier owns the merged candidate, by its own judgment; from then on it's just that group's candidate.

One persistent verifier per group with ≥1 candidate this round, up to 6 — dispatched via `Agent` the first time its group has any candidate at all (not necessarily iteration 1), resumed via `SendMessage` every time after. Its job is to **refute** each candidate in its group, with full tool access to the real target file and run artifacts — never trusting the reviewer's description. The parent hands it no context that leans toward any verdict.

Neither reviewers nor verifiers get Step 1's replace-then-drop failure handling — the 6 groups are fixed and non-interchangeable, unlike Step 1's N identical runners, and no real dispatch or resume failure among them has ever occurred to justify inventing any.

**CONFIRMED** cites the triggering input/state and the resulting error or root cause, with concrete evidence. **PLAUSIBLE** means the mechanism holds but the trigger is uncertain — state what would confirm it. **REFUTED** proves the candidate's description is wrong, or the issue's already resolved elsewhere — cite where.

REFUTED drops. CONFIRMED enters this round's `real_issues`. PLAUSIBLE defers — and only becomes real if the same reviewer independently re-raises it later and the same verifier confirms it that time; no separate tracking beyond both agents' own memory of prior rounds.

### Step 4: If real issues, fix; else count empty round

If `real_issues` is non-empty, invoke `finding-resolution` (via the `Skill` tool) once with the whole batch: `real_issues` as Issues, the target skill's stated purpose (frontmatter description + intro) as Context, `<target-skill-path>` as Target location. It drains its own queue through its 20-step flow — including anything its Read-through/Regression check turns up — and reports fixed issues (file:line + summary) against unfixed ones (issue + reason: `model-compliance` / `noise` / `non-issue`).

`empty_rounds` tracks whether the target skill directory actually changed: `diff -r` it against a `cp -r` snapshot taken at Step 4 entry. Any difference resets it to 0; no difference increments it — covering an all-`non-issue` round, a crash or silent-fail, an edit that lands then reverts, and a reference-file-only edit (still counted, whole dir snapshotted). Any real fix, even a partial one, is non-empty.

(Formatting-only drift still counts as a change — nothing here is normalized out.)

Label the outcome for summary.md: `nothing-found` (no candidates), `fixing` (something landed), `rejected-all` (candidates existed, none fixed), `partial` (some fixed, some not), `errored` (`finding-resolution` declined everything).

### Step 5: Write iteration summary.md

Write `docs/skill-optimizer-runs/<target-skill-name>/<YYYY-MM-DD>/iteration-<iteration>/summary.md`:

- **Iteration**: <iteration>
- **Runs**: succeeded count / excluded (with reason)
- **Real issues** found: title + group + CONFIRMED evidence (from that candidate's 5-field record)
- **Deferred (PLAUSIBLE) issues**: title + group + the verifier's stated uncertainty
- **Changes applied**: file:line + 1-line diff per change
- **Unfixed**: issue title + reason
- **Outcome label**
- **Diff stat**: file count / line count (against the Step 4 snapshot)
- **empty_rounds** counter

Language follows conversation; bold elements stay English.

---

## Inner Flow

Every real issue's fix runs through `finding-resolution`'s own 20-step flow — it owns the step definitions, the pendulum discipline, dispatch modes, and the 5 KISS rules. Outer Step 4 is the only place this skill invokes it.

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

- **Summary** — target skill name, N, M, total iterations, total real issues fixed, exit reason.
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
