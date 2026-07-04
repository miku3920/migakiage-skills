---
name: finding-resolution
description: Given one or more issues a reviewer has already raised (code review, spec review, skill review, PR feedback, bug report, etc.), fix each through a rigorous 16-step flow — verify facts, classify, research prior art, pendulum-swing the fix (expand then trim), adversarially KISS-check it, then apply and verify, checking for related issues along the way. Use whenever findings already exist and need fixing correctly rather than patched reactively. Do NOT use for finding issues in the first place (use a review/audit skill for that) or for open-ended exploration with no identified issue yet.
---

# Finding Resolution

Take issues a reviewer has already raised — one or many — and fix each through 16 rigorous steps: verify, classify, research, propose, swing wide then trim (adversarially checked at each swing), settle on the simplest version that satisfies the actual problem, apply, verify, and check for related issues.

**Output language:** match conversation language for narration; step headings and the fixed vocabulary below (class names, intervention types, etc.) stay English.

---

## Pace

Echo the mantra before each issue's Step 1; keep the pacing mindset everywhere else.

**Mantra:**

> Take a breath. Step by step, slowly, think each one through until it's clear. Stay calm, keep your established rhythm, neither hurried nor sluggish.

Skipping the echo because "I just did one" is the failure mode.

---

## Inputs

1. **Issues** — one or more, each with enough detail to investigate (what, where, why suspected). If any issue is too vague to act on, ask via `AskUserQuestion` before starting its flow.
2. **Context** — what the reviewed target's purpose/mission is. Derive this primarily from the review's own stated scope or intent (usually already present in how the issues were framed or the surrounding task). Only ask the user via `AskUserQuestion` if it's genuinely unclear — don't invent a fallback source (README, docs, etc.) and don't guess.
3. **Target location** — the file(s)/directory/repo being fixed.

---

## Drain Queue

Seed a queue from Inputs' Issues. For each issue in the queue: run the 16-step flow below. Don't pause between issues. Steps 15/16 may discover related issues — append them to this same queue; keep draining until it's empty.

When the queue started non-empty and ends empty, report back to the caller: for each issue, the files changed + a one-line summary of the fix (or the reason if declined as `non-issue` / crashed).

---

## 16-Step Flow

One issue at a time. Steps 7/9/11 are checks — each produces a new revised proposal (steps 8/10/12).

**Output structure (enforced):** output every step as a separately headed line `**Step N — <name>**` followed by its content. The `<name>` (e.g. "Verify facts", "Context", "Problem") is fixed vocabulary and stays English; the content under each heading follows conversation language. Anti-patterns counting as skipping:
- combining adjacent steps under one heading
- single diff fence covering multiple revisions
- omitting any step's heading
- pointing back to an earlier step instead of writing the actual content (parenthetical cross-links are fine alongside content)

The 7→12 sequence is a pendulum: 7/8 swing as far out as possible, 9/10 swing as far back in, 11/12 settle on the first-principles balance. **Narrow swings collapse step 12 back to step 6.** Narrow = swing was anchored to prior step's text (synonym-space drift), not re-derived from the Step 3 problem.

**Dispatch mode:** Step 7's N≥3 subagents run in genuine parallel — dispatch with `run_in_background: true` and wait for all notifications before merging. Steps 5, 9, and 11 each dispatch a single subagent whose result the very next step depends on — no parallelism to gain, so dispatch in the foreground (blocking) and proceed once it returns.

1. **Verify facts** — list concrete evidence: file:line citations, artifact output, reproduction steps.
2. **Context** — cross-file / cross-reference background. MUST include the target's stated purpose (from Inputs' Context), so all later steps can measure the fix against what the target exists to do.
3. **Problem** — confirm real issue + scope. MUST state the issue's relation to the target's mission: `blocker` (fix serves mission directly) / `noise` (issue is not aligned with mission, consider dropping) / `tension` (fix trades off against mission, needs careful design) / `neutral` (independent of mission). If `noise`, exit here with cited reason.
4. **Classify** — pick exactly one issue class:
   - `trivial` — typo / rename / single-word align / formatting
   - `non-issue` — existing rule/code already covers it, no change needed (exit at this step with cited reason)
   - `missing-rule` — no rule/check covers it, need to add
   - `unclear-rule` — rule/logic exists but ambiguous, rewrite in place
   - `clear-but-ignored` — rule/check exists and is clear, but was skipped anyway → forces non-prose intervention at Step 6
   - `wrong-abstraction` — logic in wrong layer / shape → delete + restructure
   - `wrong-actor` — owned by wrong function / module / step → move ownership
   - `wrong-input` — the code/agent doesn't see what it needs → change input format / required evidence

   `non-issue` exits the flow here. `trivial` skips Step 5 (research) and Steps 7-12 (pendulum + KISS + first-principles), going straight from Step 6's initial proposal to Step 13 Apply.

   **Tiebreak when multiple classes apply** (e.g. `wrong-abstraction` / `wrong-actor` / `wrong-input` can all look applicable to the same issue): check in this order — `wrong-input` (can the code/agent even see what it needs, regardless of where the logic lives?) before `wrong-actor` (is it owned by the right function/module?) before `wrong-abstraction` (is it shaped/placed correctly?). Pick the first one that applies; a class further down the list is only correct if the ones above it are already satisfied.
5. **Research** — class-specific prior art: how do mature systems / industry / existing codebases handle this issue class? Cite at least one source. Skip for `trivial` and `non-issue`. Do not let research override established principles or conventions already adopted by the target — research informs, does not replace.

   Before dispatching, state a **search domain**: one narrow, concrete technical topic that names what kind of prior art is relevant. Derive it from both Step 4's class (which kind of solution pattern to look for) and the Step 3 problem's actual mechanism (which specific technical topic). The class name alone is too generic to serve as the domain by itself. Still, don't drop the class dimension when deriving the domain — it's what points research toward the right kind of solution pattern; the Step 3 problem's mechanism supplies the other half. Embed the resulting domain in the subagent prompt as a hard scope constraint. Without this, the subagent casts a wide net across superficially-similar but mechanically-unrelated domains.

   **MUST dispatch subagent** (protects parent context from large external reads). Subagent receives the Step 3 problem (primary) + search domain + Step 2 context (reference, so it can judge whether a candidate example is structurally similar to the actual target, not just superficially similar). Prompt the subagent to: (a) read discussions / critiques / failed-replication reports, not just official docs or top-voted answers; (b) search for concrete existing open-source projects / tools that have solved a structurally similar problem, not just abstract field-level practice descriptions; (c) check whether a formal model applies to this issue class (type theory, state estimation, decision procedure with backtracking, abductive reasoning framework, etc.); (d) gather first, then classify pro/con/context, then synthesize; (e) cite sources. If (a)/(b)/(c) cover genuinely disparate ground for this search domain, the subagent may dispatch up to 3 of its own subagents in parallel (one per dimension) and synthesize their results into one report — fall back to covering all three itself if nested dispatch isn't available or the domain is narrow enough that one pass suffices. Parent verifies subagent output against established principles before adopting.
6. **Intervention type + initial proposal** — re-read the **Step 3 problem** before picking; pick exactly one intervention type, informed by Step 4 classify + Step 5 research + Step 3 mission relation (a `tension` issue needs a more careful intervention type than a `blocker`). Anti-pattern: picking based on Step 4's class label alone without re-checking it still matches the Step 3 problem's actual mechanism:
   - `prose` — add new rule/logic or rewrite existing prose/code in place
   - `procedural` — add / reorder / convert to a checklist or pipeline step (no logic rewrite)
   - `enforcement-mode` — change how strictly something is enforced (SHOULD ↔ MUST, warn ↔ throw, optional ↔ required, checklist → gate)
   - `structural` — delete + restructure / move ownership between functions, modules, or steps
   - `no-op` — rely on existing logic, document why (when classify was a near-miss for `non-issue`)
   - `soft-deprecate` — mark stale, document replacement, schedule removal

   Write initial proposal as concrete diff in ` ```diff ` fence. For `clear-but-ignored` class, `prose` is forbidden — must pick `procedural` / `enforcement-mode` / `structural`.
7. **Expand check (problem-focused)** — list every aspect of the **Step 3 problem** the fix could address. Re-derive from the problem, NOT from Step 6's draft text. Anti-pattern: synonym-space drift (listing text additions to Step 6 instead of problem aspects).

   **MUST dispatch N≥3 parallel subagents**, each from a distinct lens. One lens MUST be `mission-alignment` — that subagent receives the Step 2 context (primary anchor) + Step 3 problem + Step 6 intervention type, prompted to list aspects where the fix could serve OR undermine the target's mission. Remaining N-1 lenses drawn from user-facing / robustness / observability / maintainability / edge-case / security. Each subagent receives the Step 3 problem (primary) plus Step 6 chosen intervention type (reference, to know which type of intervention is being expanded — not to be anchored on its text), returns its lens's aspect list independently. Parent merges all lists, deduplicates, drops nothing at this step. This breaks parent's single-perspective narrow-swing tendency.
8. **Expanded version** — re-derive a new diff from Step 7's full aspect list. Deliberately bloated. Not "Step 6 + extra words".
9. **Trim check (problem-focused)** — list every aspect from Step 7 that is **not necessary to address now**. Each trim must cite the reason that aspect can be omitted (e.g. covered by future iteration, low-frequency, already mitigated elsewhere). Re-derived from problem aspects, NOT from Step 8's text.

   **MUST dispatch adversarial subagent** unless Step 7's aspect list has 2 or fewer items (trimming that short a list needs no outside pressure). This is the same sunk-cost bias as step 11: parent just expanded these aspects in Step 7/8 and defaults to keeping them — an uninvested subagent prompted to argue "this aspect should be cut" breaks that. Subagent receives the Step 3 problem (primary) plus Step 7 aspect list, Step 8 expanded version, and Step 2 context (reference, to see how far the swing went and whether an aspect fails to serve the target's mission). Parent defaults to keep; subagent must produce a concrete reason to drop, "doesn't serve the target's mission" counts as a valid reason.
10. **Trimmed version** — re-derive a new diff from Step 9's surviving aspect list. Deliberately overshot. Not "Step 8 - extra words".
11. **KISS check** — walk 5 KISS rules (see below) against step 10, each rule in three parts:
    - **Hypothesized violation** — quote a specific clause in step 10, name a concrete falsifiable violation
    - **Verification** — cite evidence from step 10 (or related sections); no evidence either way → mark "cannot verify" and treat as violated
    - **Judgment** — "violated" / "not violated" + reason (root cause if violated, refuting evidence if not). Violated clauses are step 12 restoration targets.
    Discipline: walk all 5 KISS rules fully; never collapse a rule's three parts (Hypothesized violation / Verification / Judgment) into one line ("Rule N ✓" or inline shorthand counts as skipping).

    **MUST dispatch adversarial subagent** (uninvested in step 10, no sunk cost). Subagent receives the Step 3 problem (primary) + 5 KISS rules + Step 10 trimmed diff (the target to attack) + Step 8 expanded version + Step 2 context (reference, to see what was swung out and pulled back, and to have the "business logic" Rule 2 checks against), prompted to **find violations**, not validate. For each rule, subagent attempts a hypothesized violation; parent reviews subagent's evidence and decides judgment. Parent's own walk happens in parallel as cross-check, not replacement. This breaks sunk-cost bias where parent's own KISS check rubber-stamps 5/5 not-violated.
12. **First-principles version** — diff = step 10 + step 11 restorations. Derive from "simplest change satisfying the Step 3 problem". Goes into Edit.
13. **Apply** — `Edit` directly. Don't `AskUserQuestion` for approval — this bans asking permission to proceed, not asking to resolve a genuine ambiguity (e.g. which of two files the diff targets) that step 12's diff left unresolved.
14. **Verify** — check that every change in step 12's diff (or step 6's diff if `trivial`) actually landed (any pattern tool: `grep` / `Read` / etc.); if any change is missing, re-apply via Step 13. Report each change applied (file:line + summary).
15. **Read-through** — scan the target for similar problems related to this fix. Pick a concrete check method matching the edit kind and report the tool output (don't narrate "all clear" without it; if the edit kind doesn't match any listed method, default to `Read` every section that could plausibly relate, in full):
    - added content → `Read` sections likely to contain similar patterns
    - removed content → `git diff` (or `diff` against a known-prior version) + `Read` each affected section's first content line
    - renamed identifier → `Read` each location mentioning either name
    - changed structure → `Read` sibling structural sections

    `grep` does not substitute for `Read` here — `grep` is a pointer to candidate sections; the discipline is to `Read` each candidate in full.

    For each related problem found, append it to the Drain Queue. Do not fix now — draining continues until the queue is empty.
16. **Regression check** — re-read every section related to the edit (cross-references, dependent rules, sections/functions that echo or feed into the edited one). Pick a concrete check method and report the tool output (don't narrate "no regression" without it; if the edit kind doesn't match any listed method, default to `Read` every related section in full):
    - cross-references → `Read` each section mentioning the changed anchor
    - definitions → `Read` each section using the changed term
    - schemas / interfaces → `Read` each producer and consumer
    - rule lists / anti-patterns → `Read` for contradictions with the new edit

    Verify nothing else broke: no broken cross-refs, no contradictions, no new ambiguity. For each regression found, append it to the Drain Queue. Do not fix now — draining continues until the queue is empty.

### What "KISS" means here (for step 11)

KISS = Keep It Simple, Stupid. Go back to first principles, keep only what's actually needed, stay true to the business facts, and hold up long-term. Applies to step 12 final; step 10 overshoot is a pendulum tool.

Five rules:

1. **Any simplification must first satisfy first principles** — return to essence
2. **No simplification may violate business logic** — business facts are not fat
3. **A good system must remain continuously explainable, verifiable, and maintainable**
4. **Real simplification cuts understanding, verification, and maintenance cost together**
5. **If a fix makes the future harder, it isn't KISS**
