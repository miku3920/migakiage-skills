---
name: skill-optimizer
description: Iteratively optimize an existing skill by running N parallel subagents against a test input, comparing artifacts across runs, identifying real issues (failures reproduced in ≥ N/3 of runs, or a mission-fit gap ≥ N/3 reviewers independently cite) and fixing the skill via a 16-step revision flow. Loops until M consecutive empty rounds = convergence. Use whenever the user asks to iterate, optimize, refine, harden, improve, validate, or empirically debug an existing skill against a real task — even if they only say "make this skill better" or "find bugs in this skill". Do NOT use for creating a new skill from scratch (use skill-creator instead).
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
3. **N** — parallel runner subagents per round, ≥ 3 (below 3 the `≥ N/3` threshold collapses; at N=3 Suspected also collapses). Step 2 additionally dispatches N reviewer subagents (one per runner, a separate headcount from this N) — actual dispatch cost per iteration is 2N, not N.
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
  step 2: dispatch N reviewer subagents (one per runner) to independently read each run's full output directory; merge into cross-run artifact comparison
  step 3: classify issues (real if ≥ N/3 runs show the same observed failure, or ≥ N/3 reviewers independently cite the same mission-fit gap)
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

When complete, your final text IS the return value — report any harness friction (tool blocked, dispatch failed, etc.).
```

Wait for all background task notifications. Before calling any wait/poll/sleep-like tool (`ScheduleWakeup`, a `TaskOutput` status-check loop, a `Bash` sleep loop) to wait on this dispatch, ask: is this for work I background-dispatched in this same flow? If yes — stop, do not call it; the harness delivers a `task-notification` automatically when each completes, and calling one anyway is the failure mode, not a safety margin. This does not apply to genuinely external/untracked work — `ScheduleWakeup` remains the correct tool there. A single status check is a legitimate escalation only after ending at least one full turn without the notification arriving; more than one check within the same turn is a loop, not an escalation, and is not legitimate.

If a subagent fails to produce artifacts (no notification / crash / dispatch error / wrote nothing to its run directory), try one replacement. If that also fails, drop it. Let K = total drops this round. If (N-K) ≥ 3 carry on. If (N-K) < 3 abort (see Stopping Conditions).

### Step 2: Cross-run artifact comparison

**Reviewer role**: a second, independent subagent role, one per runner (N reviewers for N runners — a separate headcount from Step 1's N). Read-only; Step 1's existing "wait for all background task notifications" barrier already guarantees every runner is done and flushed before Step 2 starts, so no separate completion signal is needed here.

**Investigation mandate, set before the reviewer reads anything**: investigate only deviations that would directly change whether the target skill achieves the mission stated verbatim in its own `SKILL.md` frontmatter. A deviation is in scope only if it changes a run's mission-relevant outcome through a direct mechanism, not a multi-step rationalization chain (e.g. "cosmetic issue → erodes trust → undermines mission" is out of scope; state the direct mechanism or drop it). A deviation ruled out of scope under this mandate is not written up as a finding, but note it in one line under "Screened out (reason)" in the observations table below, so parent's spot-check (see below) can sample it the same way it samples cited findings.

Stated positively, not just as an exclusion: ask whether the target skill's own design — its methodology, coverage, or approach — actually gives it the best chance of achieving the mission stated in its frontmatter, not only whether each individually-named phase, gate, or check works as documented. A design choice that structurally limits the mission (a gap no phase covers, a check that verifies the wrong thing, a step whose output nothing downstream uses) is in scope even where every named component technically does what it claims. The same evidence discipline applies as anywhere else in this section: file:line in a `run-<i>/` artifact if the gap actually manifested there, or the purpose-quote-plus-cited-run bar above if it can't be pinned to a run artifact but genuinely occurred — never a bare citation to the target skill's own `SKILL.md` with no run behind it.

**Dispatch**: same mode as Step 1 — `run_in_background: true`, parallel, wait for all notifications before merging. Same failure handling as Step 1's replacement-then-drop policy and K/(N-K) ≥ 3 math; a reviewer that permanently fails records its run under this section's "Excluded runs (reason)" column with reason `reviewer-failed` — that column now holds either a runner-side or reviewer-side failure reason, distinguish by the reason string.

**Reviewer inputs** (three, reference-only — reviewer never executes the target skill):
1. The full `run-<i>/` directory — every file the runner produced, not a guessed filename or assumed structure.
2. The test input from Input #2 (parent already holds this).
3. The target skill's `SKILL.md` — schema/expected-fields reference, so the reviewer judges against the target skill's own definition rather than the runner's self-interpretation of it (a different purpose than Step 1's execution-read of the same file).

While reading input #3, also judge whether the target skill's own definition serves the purpose stated in its own frontmatter. A valid observation needs the same evidence discipline as any other finding in this section: quote the purpose claim verbatim from the target `SKILL.md` (never the reviewer's own paraphrase of "what it probably means"), and ground it in an actual run (cite `run-<i>/` — not a hypothetical). If a deviation is already citable by file:line, it stays a file:line finding — this channel exists only for what genuinely can't be pinned to a line, not as a way around that bar. E.g.: target skill states its purpose as "catch races before production"; Phase 4's gate re-checks what Phase 2 already checks while no phase covers message-reordering races, and run-2's target codebase had an actual reordering bug every phase missed — that's a valid observation; "the phasing feels bureaucratic" with no quoted purpose and no cited run is not.

**Reviewer output**: for each claim about that run's output, cite the specific file:line it came from. If the run directory, target `SKILL.md`, or Input #2 is empty, malformed, or missing, degrade gracefully and report `insufficient evidence` for that gap rather than guessing. Artifact that can't be cited by file:line → mark `[uncited]`, don't fabricate a line number.

This includes exit-gate pass/fail claims: the reviewer checks each gate defined in the target skill's `SKILL.md` against the run's actual artifacts and cites the result, the same way as any other deviation — exit gates are part of "the target skill's own definition" the reviewer was already given input #3 to judge against. If the target skill defines no gates at all, report `N/A` for that run — this is a valid state, not a data gap, and must not be reported as `insufficient evidence`. If a gate is described only informally/in prose (no discrete checklist), and the reviewer cannot ground a pass/fail judgment in a specific citable line, mark it `[uncited]` rather than synthesizing a judgment call. A gate recurring as failed across ≥ N/3 runs is itself a candidate issue — route it into Cross-run's "Candidate issues for classification" the same way a recurring deviation would be, subject to the same citation bar: an `[uncited]` gate failure cannot supply the file:line evidence Step 3's `Real` classification requires, same as an `[uncited]` deviation. A one-off single-run gate failure is not, by itself, evidence of a target-skill defect.

Parent merges the N reviewer reports (not runner self-reports) to compare findings; before Step 3 accepts a candidate as `Real` (≥ N/3 recurrence), parent spot-checks that candidate's cited evidence directly against the actual files — file:line for a mechanical deviation, or that the quoted purpose claim really appears verbatim in the target `SKILL.md` and the cited run scenario really occurred for a mission-fit observation. Parent also spot-checks a sample of this round's "Screened out" entries the same way — re-reading the cited deviation against the target skill's own frontmatter and confirming the direct-mechanism test actually holds. If it doesn't hold (the entry was wrongly screened out), parent routes it into Cross-run's "Candidate issues for classification" the same way a recurring deviation would be, subject to the same ≥ N/3 recurrence bar as any other candidate. Stay observation-driven.

Write `observations.md` (language follows conversation; bold elements stay English):

**Per-run** (table):
- Run ID
- Artifacts produced
- Exit gates pass/fail (N/A if none) — reviewer-verified against the target skill's own definition; runner self-report is no longer an accepted source for this column, same standard as the deviations column below
- Reviewer-cited deviations (file:line) — runner self-report is no longer an accepted source for this column
- Screened out (reason) — deviations investigated but excluded under the Investigation mandate; one line each, with the direct-mechanism test cited; distinct from "Excluded runs" below (whole-run failures, not scoping decisions)
- Harness friction
- Excluded runs (reason)

**Cross-run** (free prose feeding Step 3):
- Shared findings (which N)
- Severity divergence
- Finding count + scope spread
- Recurring issues from prior summary.md (skip on iteration 1)
- Candidate issues for classification
- Mission-fit observations (quoted purpose claim + cited run scenario, kept separate from Candidate issues above — different evidence shape)

### Step 3: Classify issues

Echo the Pace mantra verbatim (see `## Pace`) before continuing.

For each candidate issue (from observations.md Cross-run):

- **Real issue**: ≥ N/3 runs with reviewer-cited artifact evidence (file:line) — fix.
- **Suspected**: < N/3 runs — defer. At N=3 this never fires.
- **Hypothetical (file-imagined)**: runner's text-only claim with no reviewer citation backing it — drop.
- **Real mission-fit issue**: ≥ N/3 reviewers independently name the same quoted purpose claim + cite the same run scenario — fix, routed through `finding-resolution` the same as a real issue (it decides severity through whatever path it already uses for any other finding — no new vocabulary here). This is a distinct pipeline point from `finding-resolution`'s own Step 7 mission-alignment lens, which only evaluates a fix's direction after a finding is already flagged — the two don't compete, since this step is what supplies the candidate in the first place.
- **Suspected mission-fit**: < N/3 reviewers — defer, same as Suspected above.
- **Hypothetical mission-fit (reviewer-imagined)**: a reviewer's claim with no verbatim purpose quote or no cited run scenario — drop, same as Hypothetical above.

Examples (N=3, threshold ≥ 1; any artifact-backed hit counts):
- 1/3 runs skip same instruction with artifact evidence → real
- 1/3 self-report tool blocked, no artifact → hypothetical (drop)
- "method count 7 vs 6" from reading skill, no failure → hypothetical
- 1/3 reviewers name the skill's own stated purpose + cite a real run where the current phase structure let a matching bug through → real mission-fit issue

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
- **Real issues** found: title + ≥ N/3 evidence (file:line for mechanical issues; quoted purpose claim + cited run scenario for mission-fit issues)
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
