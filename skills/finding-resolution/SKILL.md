---
name: finding-resolution
description: Given one or more issues a reviewer has already raised (code review, spec review, skill review, PR feedback, bug report, etc.), fix each through a rigorous 20-step flow — explore, verify facts, reproduce, attribute root cause, confirm context and problem, classify, research prior art, pendulum-swing the fix (expand then trim), adversarially KISS-check it, then apply, verify, independently review, and check for related issues along the way. Use whenever findings already exist and need fixing correctly rather than patched reactively. Do NOT use for finding issues in the first place (use a review/audit skill for that) or for open-ended exploration with no identified issue yet.
---

# Finding Resolution

Take issues a reviewer has already raised — one or many — and fix each through 20 rigorous steps: explore, verify facts, reproduce, attribute root cause, confirm context and problem, classify, research, propose, swing wide then trim (adversarially checked at each swing), settle on the simplest version that satisfies the actual problem, apply, verify, independently review, and check for related issues.

**Output language:** match conversation language for narration; step headings and the fixed vocabulary below (class names, intervention types, etc.) stay English.

---

## Pace

Echo the mantra before each issue's **Explore** step; keep the pacing mindset everywhere else.

**Mantra:**

> Take a breath. Step by step, slowly, think each one through until it's clear. Stay calm, keep your established rhythm, neither hurried nor sluggish.

Skipping the echo because "I just did one" is the failure mode.

---

## Inputs

1. **Issues** — one or more, each with enough detail to investigate (what, where, why suspected). If any issue is too vague to act on, ask via `AskUserQuestion` before starting its flow.
2. **Context** — what the reviewed target's purpose/mission is. Read the review's own stated scope or intent — how the issues were framed, the surrounding task — to derive this. Only ask the user via `AskUserQuestion` if, having read that framing, no purpose/intent statement is there to find, or what's there is unresolvably ambiguous or self-contradictory — don't invent a fallback source (README, docs, etc.) and don't guess.
3. **Target location** — the file(s)/directory/repo being fixed. Try deriving it from Issues' own "where" first (Input #1 already requires it); treat that as provisional, not final — if **Classify**'s classification later shows the fix site isn't the citation site (e.g. `wrong-actor`), revisit it there. If nothing can be derived, or what's derivable names multiple candidate locations with no way to pick one, ask via `AskUserQuestion` — don't invent a fallback and don't guess, same don't-invent-don't-guess principle as Input #2.

---

## Drain Queue

Seed a queue from Inputs' Issues, each entry carrying an issue identifier (e.g. `issue 1`, `issue 2`) — a batch's cross-issue references (e.g. "issue 2's Attribution") always use this identifier alongside the step name, since within a multi-issue batch every issue has its own copy of every step. For each issue in the queue: run the 20-step flow below, starting fresh from **Explore** — don't inherit another queued issue's **Explore** or any other step's output even if they target the same file. Don't pause between issues. **Read-through**/**Regression check** may discover related issues — append them to this same queue, each running the full flow independently from its own **Explore**; keep draining until it's empty.

When the queue started non-empty and ends empty, report back to the caller: for each issue, the files changed + a one-line summary of the fix (or the reason if declined as `model-compliance` / `noise` / `non-issue` / crashed) — plus, as a separate list, any unconnected observations **Read-through**/**Regression check** noted but didn't fix (not in scope, surfaced for the caller to decide).

---

## 20-Step Flow

One issue at a time. **Expand check** / **Trim check** / **KISS check** are checks — each produces a new revised proposal (**Expanded version** / **Trimmed version** / **KISS version**).

**Output structure (enforced):** output every step as a separately headed line `**Step N — <name>**` followed by its content, where N is that step's position in the numbered list above (e.g. `**Step 6 — Attribution**`). The `<name>` (e.g. "Fact check", "Context", "Problem") is fixed vocabulary and stays English; the content under each heading follows conversation language. This number is a local position marker for the header you're writing, not a cross-reference — every cross-reference elsewhere in this flow still uses the bold step name alone, never a bare number. Write each step's header *before* doing that step's work — its tool calls, subagent dispatches, or analysis — not after the work is already done, with the header retrofitted around a retroactive summary. Anti-patterns counting as skipping:
- combining adjacent steps under one heading
- single diff fence covering multiple revisions
- omitting any step's heading
- pointing back to an earlier step instead of writing the actual content (parenthetical cross-links are fine alongside content)
- writing a step's header after already doing that step's work, instead of announcing it first

**Required headers (checklist):** every step in the range from **Explore** up to the point you're about to write or exit at must already have its own header — no step skips this except where named here:
- a `model-compliance` exit lands right after **Attribution** — everything from **Explore** through **Attribution**.
- a `noise` exit lands right after **Problem** — everything from **Explore** through **Problem**.
- a `non-issue` exit lands right after **Classify** — everything from **Explore** through **Classify**.
- `trivial` is decided at the same point as `non-issue` (**Classify**) but continues instead of exiting: it reaches **Initial proposal** with everything from **Explore** through **Classify** headed, then skips **Research** and skips **Expand check** through **KISS version** entirely — going straight to **Apply**.
- everything else reaches **Initial proposal** with everything from **Explore** through **Research** headed.

If **Problem**'s mission-relation conclusion changes mid-analysis (e.g. an issue looks headed for a `noise` exit, but **Problem**'s own analysis lands on `blocker`/`tension`/`neutral` instead), the required range above recomputes from that new conclusion — proceed to **Attribution** and its own header, don't treat the earlier expectation as already satisfying anything past **Problem**. Each issue in a Drain Queue batch resets this independently — a prior issue's headers never carry over.

**The Iron Law:** A STEP'S HEADER IS WRITTEN BEFORE ANY OF THAT STEP'S WORK. NO EXCEPTIONS.

There is no fixing this after the fact — the moment work happens before its header, that ordering is already permanent in the transcript. A "redo" doesn't undo it, it only adds more content after an out-of-order violation that already stands. Get it right the first time.

The **Expand check** → **KISS version** sequence is a pendulum: **Expand check**/**Expanded version** swing as far out as possible, **Trim check**/**Trimmed version** swing as far back in, **KISS check**/**KISS version** settle on the first-principles balance. **Narrow swings collapse KISS version back to Initial proposal.** Narrow = swing was anchored to the prior step's text (synonym-space drift), not re-derived from the **Problem** step.

**Dispatch mode:** **Expand check**'s N≥3 subagents run in genuine parallel — dispatch with `run_in_background: true` and wait for all notifications before merging. Before calling any wait/poll/sleep-like tool (`ScheduleWakeup`, a `TaskOutput` status-check loop, a `Bash` sleep loop) to wait on this dispatch, ask: is this for work I background-dispatched in this same flow? If yes — stop, do not call it; the harness delivers a `task-notification` automatically when each completes, and calling one anyway is the failure mode, not a safety margin. This does not apply to genuinely external/untracked work (a CI run, a remote queue, a human reply) — `ScheduleWakeup` remains the correct tool there. A single status check is a legitimate escalation only after ending at least one full turn without the notification arriving; more than one check within the same turn is a loop, not an escalation, and is not legitimate. **Research**, **Trim check**, and **KISS check** each dispatch a single subagent whose result the very next step depends on — no parallelism to gain, so dispatch in the foreground (blocking) and proceed once it returns.

1. **Explore** — read whatever the issue and Target location point at before forming any conclusion: the flagged file/section, and anything it visibly depends on. No summary or narration needed under this heading — the tool calls you actually make (`Read` / `Grep` / `Bash`) are this step's evidence; don't write a paraphrase of them. This exists to stop **Fact check**/**Red verify** from starting on an assumed rather than an actually-read target.
2. **Fact check** — list concrete evidence for the issue as raised: file:line citations, existing artifact output, anything already on record. This step's only job is "what can I point at right now" — it does not build anything new (that's **Red verify**) and does not decide why it happened (that's **Attribution**).
3. **Red verify** — this step's only job is proof, not restatement: if the issue has executable behavior, build a re-runnable reproduction and demonstrate it currently fails. Match the method to what's actually changing — a code change gets a real unit test or script, written and executed; a prompt, instruction, or skill/spec change (anything a reader or agent would ever consume to act on) gets its identical, unparaphrased case text handed to an independent agent, uninvested and seeing only the reproduction scenario, held to the same standard as this flow's other adversarial dispatches; a genuine mix (a config file, a change touching both) takes whichever of the two yields a citable, re-run-and-observe result, with the pick and its reason stated. A weak assertion, an unexecuted test, a test scoped to the wrong thing, a test retrofitted after the fact, or reasoning about what a run would probably show — none of these clear this step. **Green verify** re-runs this same artifact later; don't build a second one there.

   Judgment-based behavior still counts as executable here: verifying a procedure was followed correctly, or an agent's response to ambiguous instruction text in a specific scenario, has no deterministic assert to run, but it's still an agent-executable scenario — "pass" means a specific, named signal actually occurred (a specific tool call fired, a specific file ended in a specific state), never "the agent judged it correct." Nothing about the issue's subject matter exempts it from this step: if it's text a reader or agent would ever consume to act on, a scenario can always be built. The one substitute for a fresh run is prior real operation — simulating a decision point stands in for history only if that exact point has already fired and been observed at least once, with the concrete prior instance cited, not just that it's reachable in principle; zero real executions to date doesn't qualify. (An abort-condition clause that has genuinely triggered before can be re-simulated against that same triggering state; a newly-written abort-condition that's never fired yet doesn't qualify for this substitution, and reasoning about it hypothetically is an unverified guess, not a finding.)

   Skip this step only when the issue has no executable behavior at all — `trivial` is not a reason to skip it; check **Classify**'s skip-list, which doesn't include this step. This step runs on the issue's own case alone — it needs nothing from **Context** or **Problem** to start.
4. **Context** — cross-file / cross-reference background. MUST include the target's stated purpose (from Inputs' Context), so all later steps can measure the fix against what the target exists to do.
5. **Problem** — confirm real issue + scope. MUST state the issue's relation to the target's mission: `blocker` (fix serves mission directly) / `noise` (issue is not aligned with mission, consider dropping) / `tension` (fix trades off against mission, needs careful design) / `neutral` (independent of mission). If `noise`, exit here with cited reason.
6. **Attribution** — replay the flagged instance through the target's *current*, unmodified logic (the **Red verify** reproduction). If current logic, as written, produces the correct outcome for this exact input, and only failed to prevent the flagged instance because a bounded list/enumeration it depends on hadn't yet included this specific value — that's **source-variance** (the logic itself is sound; its checked input — e.g. an LLM's own generated output — has irreducible ongoing variation no such list can ever fully close). If current logic has no branch/condition that would catch this input regardless of content — that's **target-defect** (a real gap in the logic itself). For a target with no such enumerable-content-dependent check at all (a static code bug, a fixed spec's ambiguous wording), this resolves trivially to target-defect — don't spend more than this sentence on it. This attribution is not itself a `non-issue` verdict: `non-issue` requires the value to already be covered, which source-variance's own definition rules out (that's why the instance was flagged) — a source-variance finding still goes through **Classify** normally, typically landing on `no-op` (**Initial proposal**) to document the residual rather than chasing full coverage of an irreducible source.

   When the finding concerns whether an actor complied with an instruction (not a static code gap), target-defect further requires that compliance be checkable by a party other than that actor — e.g. a tool-call transcript, a reference list, an executable test's own result, a diff, a log — produced as a byproduct of how the target already operates, not self-attestation the actor alone controls. If no such independently-checkable evidence exists or could exist for this specific claim — the only possible evidence is the actor's own account — this is **model-compliance**: exit the flow here with cited reason, the same mechanism `noise` and `non-issue` use elsewhere; do not add any marker to the target file to remember this verdict — the Drain Queue's final report to the caller (above) already carries that record, the same way it does for every other exit in this flow. This step needs **Red verify**'s reproduction to have something to replay, and **Context**/**Problem**'s mission information to correctly separate "actor didn't comply" from "matches the target's actual design intent" — it cannot run before either, and its model-compliance judgment specifically uses **Problem**'s mission-relation conclusion as an input, not just a preceding step to point at.
7. **Classify** — pick exactly one issue class:
   - `trivial` — typo / rename / single-word align / formatting
   - `non-issue` — existing rule/code already covers it, no change needed (exit at this step with cited reason; if **Attribution** found `source-variance`, cite that finding directly here rather than re-deriving coverage independently)
   - `missing-rule` — no rule/check covers it, need to add
   - `unclear-rule` — rule/logic exists but ambiguous, rewrite in place
   - `clear-but-ignored` — rule/check exists and is clear, but was skipped anyway → forces non-prose intervention at **Initial proposal**
   - `wrong-abstraction` — logic in wrong layer / shape → delete + restructure
   - `wrong-actor` — owned by wrong function / module / step → move ownership
   - `wrong-input` — the code/agent doesn't see what it needs → change input format / required evidence

   `non-issue` exits the flow here. `trivial` skips **Research** and **Expand check** through **KISS version**, going straight from **Initial proposal** to **Apply**.

   **Tiebreak when multiple classes apply** (e.g. `wrong-abstraction` / `wrong-actor` / `wrong-input` can all look applicable to the same issue): check in this order — `wrong-input` (can the code/agent even see what it needs, regardless of where the logic lives?) before `wrong-actor` (is it owned by the right function/module?) before `wrong-abstraction` (is it shaped/placed correctly?). Pick the first one that applies; a class further down the list is only correct if the ones above it are already satisfied.
8. **Research** — class-specific prior art: how do mature systems / industry / existing codebases handle this issue class? Cite at least one source. Skip for `trivial` and `non-issue`. Do not let research override established principles or conventions already adopted by the target — research informs, does not replace.

   Before dispatching, state a **search domain**: one narrow, concrete technical topic that names what kind of prior art is relevant. Derive it from both **Classify**'s class (which kind of solution pattern to look for) and **Problem**'s actual mechanism (which specific technical topic). The class name alone is too generic to serve as the domain by itself. Still, don't drop the class dimension when deriving the domain — it's what points research toward the right kind of solution pattern; **Problem**'s mechanism supplies the other half. Embed the resulting domain in the subagent prompt as a hard scope constraint. Without this, the subagent casts a wide net across superficially-similar but mechanically-unrelated domains.

   **MUST dispatch subagent** (protects parent context from large external reads). Subagent receives the **Problem** step (primary) + search domain + **Context** (reference, so it can judge whether a candidate example is structurally similar to the actual target, not just superficially similar). Prompt the subagent to: (a) read discussions / critiques / failed-replication reports, not just official docs or top-voted answers; (b) search for concrete existing open-source projects / tools that have solved a structurally similar problem, not just abstract field-level practice descriptions; (c) check whether a formal model applies to this issue class (type theory, state estimation, decision procedure with backtracking, abductive reasoning framework, etc.); (d) gather first, then classify pro/con/context, then synthesize; (e) cite sources. If (a)/(b)/(c) cover genuinely disparate ground for this search domain, the subagent may dispatch up to 3 of its own subagents in parallel (one per dimension) and synthesize their results into one report — fall back to covering all three itself if nested dispatch isn't available. Parent verifies subagent output against established principles before adopting.
9. **Initial proposal** — re-read the **Problem** step before picking; pick exactly one intervention type, informed by **Classify** + **Research** + **Problem**'s mission relation (a `tension` issue needs a more careful intervention type than a `blocker`). Anti-pattern: picking based on **Classify**'s class label alone without re-checking it still matches **Problem**'s actual mechanism:
   - `prose` — add new rule/logic or rewrite existing prose/code in place
   - `procedural` — add / reorder / convert to a checklist or pipeline step (no logic rewrite)
   - `enforcement-mode` — change how strictly something is enforced (SHOULD ↔ MUST, warn ↔ throw, optional ↔ required, checklist → gate)
   - `structural` — delete + restructure / move ownership between functions, modules, or steps
   - `no-op` — rely on existing logic, document why (when **Classify** was a near-miss for `non-issue`)
   - `soft-deprecate` — mark stale, document replacement, schedule removal

   Write initial proposal as concrete diff in ` ```diff ` fence. For `clear-but-ignored` class, `prose` is forbidden — must pick `procedural` / `enforcement-mode` / `structural`.
10. **Expand check** — list every aspect of the **Problem** step the fix could address. Re-derive from the problem, NOT from **Initial proposal**'s draft text. Anti-pattern: synonym-space drift (listing text additions to **Initial proposal** instead of problem aspects).

    **MUST dispatch N≥3 parallel subagents**, each from a distinct lens. One lens MUST be `mission-alignment` — that subagent receives **Context** (primary anchor) + **Problem** + **Initial proposal**'s intervention type, prompted to list aspects where the fix could serve OR undermine the target's mission. Remaining N-1 lenses drawn from user-facing / robustness / observability / maintainability / edge-case / security. Each subagent receives **Problem** (primary) plus **Initial proposal**'s chosen intervention type (reference, to know which type of intervention is being expanded — not to be anchored on its text), returns its lens's aspect list independently. Parent merges all lists, deduplicates, drops nothing at this step. This breaks parent's single-perspective narrow-swing tendency.
11. **Expanded version** — re-derive a new diff from **Expand check**'s full aspect list. Deliberately bloated. Not "**Initial proposal** + extra words".
12. **Trim check** — list every aspect from **Expand check** that is **not necessary to address now**. Each trim must cite the reason that aspect can be omitted (e.g. covered by future iteration, low-frequency, already mitigated elsewhere). Re-derived from problem aspects, NOT from **Expanded version**'s text.

    **MUST dispatch adversarial subagent** unless **Expand check**'s aspect list has 2 or fewer items (trimming that short a list needs no outside pressure). This is the same sunk-cost bias as **KISS check**: parent just expanded these aspects in **Expand check**/**Expanded version** and defaults to keeping them — an uninvested subagent prompted to argue "this aspect should be cut" breaks that. Subagent receives **Problem** (primary) plus **Expand check**'s aspect list, **Expanded version**, and **Context** (reference, to see how far the swing went and whether an aspect fails to serve the target's mission). Parent defaults to keep; subagent must produce a concrete reason to drop, "doesn't serve the target's mission" counts as a valid reason.
13. **Trimmed version** — re-derive a new diff from **Trim check**'s surviving aspect list. Deliberately overshot. Not "**Expanded version** - extra words".
14. **KISS check** — walk 5 KISS rules (see below) against **Trimmed version**, each rule in three parts:
    - **Hypothesized violation** — quote a specific clause in **Trimmed version**, name a concrete falsifiable violation
    - **Verification** — cite evidence from **Trimmed version** (or related sections); no evidence either way → mark "cannot verify" and treat as violated
    - **Judgment** — "violated" / "not violated" + reason (root cause if violated, refuting evidence if not). Violated clauses are **KISS version** restoration targets.
    Discipline: walk all 5 KISS rules fully; never collapse a rule's three parts (Hypothesized violation / Verification / Judgment) into one line ("Rule N ✓" or inline shorthand counts as skipping).

    **MUST dispatch adversarial subagent** (uninvested in **Trimmed version**, no sunk cost). Subagent receives **Problem** (primary) + 5 KISS rules + **Trimmed version** (the target to attack) + **Expanded version** + **Context** (reference, to see what was swung out and pulled back, and to have the "business logic" Rule 2 checks against), prompted to **find violations**, not validate. For each rule, subagent attempts a hypothesized violation; parent reviews subagent's evidence and decides judgment. Parent's own walk happens in parallel as cross-check, not replacement. This breaks sunk-cost bias where parent's own KISS check rubber-stamps 5/5 not-violated.
15. **KISS version** — diff = **Trimmed version** + **KISS check** restorations. Derive from "simplest change satisfying the **Problem** step". Goes into **Apply**.
16. **Apply** — `Edit` directly. Don't `AskUserQuestion` for approval — this bans asking permission to proceed, not asking about a gap that's actually in the diff being applied (**KISS version**'s, or **Initial proposal**'s if `trivial`) — checkable by re-reading that diff, not this agent's own uncertainty about a diff that's actually complete.
17. **Green verify** — check that every change in **KISS version**'s diff (or **Initial proposal**'s diff if `trivial`) actually landed (any pattern tool: `grep` / `Read` / etc.); if any change is missing, re-apply via **Apply**. If **Red verify** constructed a reproduction, re-run that same artifact now (not a new one) and confirm it passes, citing the actual output. Any supplementary case built anywhere within this same issue's **Green verify** through **Regression check** to strengthen confidence — beyond the one artifact **Red verify** may have constructed — is held to the same red-then-green standard: establish red using **KISS version**'s diff's own pre-fix wording (an actual test run for a code target; that exact text fed to an independent agent for a prompt/instruction target; for anything else, whichever of the two **Red verify** would have picked), not by reasoning about what it would have shown, then establish green the same way against the post-fix wording. A new issue **Read-through**/**Regression check** discovers is unaffected by this — it seeds its own Drain Queue entry and runs the whole flow independently from its own **Explore**. Report each change applied (file:line + summary) and the reproduction's result.
18. **Review** — dispatch an independent subagent (uninvested, no sunk cost in the fix just applied) to check whether the applied change actually resolves the **Problem** step, not just whether **Green verify**'s artifact turned green. Subagent receives **Problem** (primary), the actual post-**Apply** target content (not the diff — read the real file), and **Context**; prompted to answer: does this fix address the problem as stated, or does it satisfy the letter of the reproduction while missing the actual issue? Report the subagent's verdict; if it finds a gap, that gap seeds a new Drain Queue entry with its own **Explore** — this step doesn't re-open **Apply** for the current issue.
19. **Read-through** — scan the target for similar problems related to this fix. Before checking, state the full list of candidate files/sections to inspect — a list assembled after the fact, or narrowed mid-check, doesn't satisfy this. Then pick a concrete check method matching the edit kind and report the tool output per candidate (don't narrate "all clear" without it; if the edit kind doesn't match any listed method, default to `Read` every section that could plausibly relate, in full):
    - added content → `Read` sections likely to contain similar patterns
    - removed content → `git diff` (or `diff` against a known-prior version) + `Read` each affected section's first content line
    - renamed identifier → `Read` each location mentioning either name
    - changed structure → `Read` sibling structural sections

    `grep` does not substitute for `Read` here — `grep` is a pointer to candidate sections; the discipline is to `Read` each candidate in full. Nor does an `Edit`/`Write` tool message about a *different* file: "no need to Read it back" covers only the exact file just written, not anything else you're checking in this step.

    The bullets above are search techniques, not a connectedness test — finding a candidate that way doesn't by itself make it related. It counts as a related problem only if you can cite the actual textual relationship at that location (file:line) — e.g. a real mention of the changed term/anchor, a real shared pattern. Something a search technique surfaced but that has no such relationship isn't related; note it as an unconnected observation instead (Drain Queue's final report carries these separately).

    **MUST dispatch one adversarial subagent, unconditionally — no carve-out, including `trivial`** (both observed failures happened here; one was on the `trivial` path). Same rationale as **KISS check**. Subagent receives: **Problem**, **KISS version**/**Apply**'s diff, **Context**, Target location — not the parent's own candidate list or Read outputs, so it searches independently, bound by the same Read discipline and the same file:line-connection standard above. Treat any cited finding from either side as real; if both cite the same one, append once.

    For each related problem found, append it to the Drain Queue. Do not fix now — draining continues until the queue is empty.
20. **Regression check** — re-read every section related to the edit (cross-references, dependent rules, sections/functions that echo or feed into the edited one). Before checking, state the full list of candidate files/sections to inspect, covering every file the edit touched, not just the primary one — a list assembled after the fact, or narrowed mid-check, doesn't satisfy this. Then pick a concrete check method and report the tool output per candidate (don't narrate "no regression" without it; if the edit kind doesn't match any listed method, default to `Read` every related section in full):
    - cross-references → `Read` each section mentioning the changed anchor
    - definitions → `Read` each section using the changed term
    - schemas / interfaces → `Read` each producer and consumer
    - rule lists / anti-patterns → `Read` for contradictions with the new edit

    `grep` does not substitute for `Read` here either — a `grep` match list is not evidence of "no contradictions," and a `grep` whose scope omits one of the edited files proves nothing about that file. Nor does an `Edit`/`Write` tool message about one file extend to any other file or section you're checking here. `Read` each candidate section in full, in every file the edit touched.

    Same file:line-connection standard as **Read-through**'s: the bullets above are search techniques, not proof of a regression — cite the actual textual relationship at that location, or it's an unconnected observation, not a regression.

    Same mandatory-unconditional adversarial-subagent mechanism as **Read-through**'s, applied to regression-checking instead of related-problem-finding.

    Verify nothing else broke: no broken cross-refs, no contradictions, no new ambiguity. For each regression found, append it to the Drain Queue. Do not fix now — draining continues until the queue is empty.

### What "KISS" means here (for **KISS check**)

KISS = Keep It Simple, Stupid. Go back to first principles, keep only what's actually needed, stay true to the business facts, and hold up long-term. Applies to **KISS version** final; **Trimmed version** overshoot is a pendulum tool.

Five rules:

1. **Any simplification must first satisfy first principles** — return to essence
2. **No simplification may violate business logic** — business facts are not fat
3. **A good system must remain continuously explainable, verifiable, and maintainable**
4. **Real simplification cuts understanding, verification, and maintenance cost together**
5. **If a fix makes the future harder, it isn't KISS**
