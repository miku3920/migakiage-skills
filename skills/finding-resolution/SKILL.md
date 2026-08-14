---
name: finding-resolution
description: Fix issues a reviewer has already raised (code review, spec review, skill review, PR feedback, bug report, etc.) through this file's rigorous, adversarially-verified 20-step verify-then-fix flow. Use whenever findings already exist and need fixing correctly rather than patched reactively. Do NOT use for finding issues in the first place (use a review/audit skill for that) or for open-ended exploration with no identified issue yet.
---

# Finding Resolution

A reviewer has already raised one or more issues. Run each through the **20-Step Flow** below, starting at **Explore**.

**Output language:** narration follows the conversation language; step headings and the fixed vocabulary below (class names, etc.) stay in English.

---

## Pace

Echo the mantra before each issue's **Explore** step; keep the pacing mindset everywhere else.

**Mantra:**

> Take a breath. Step by step, slowly, think each one through until it's clear. Stay calm, keep your established rhythm, neither hurried nor sluggish.

Skipping the echo because "I just did one" is the failure mode.

---

## Inputs

1. **Issues** — one or more, each with enough to investigate: what's wrong, where, why it's suspected. Too vague to act on? Ask via `AskUserQuestion` before that issue's flow starts.
2. **Context** — the reviewed target's purpose or mission. Read it out of the review's own stated scope or intent. Ask the user via `AskUserQuestion` only when no purpose statement exists to find, or what exists is unresolvably ambiguous — never invent a substitute, never guess one.
3. **Target location** — the file, directory, or repo being fixed. Derive it from **Issues**' own "where." Nothing derivable, or several candidates with no way to choose? Ask via `AskUserQuestion`, under the same don't-invent-don't-guess rule as Input #2.

---

## Drain Queue

Every issue from Inputs' **Issues** seeds one queue entry, carrying an identifier (`issue 1`, `issue 2`, ...). When a step needs to reference another issue's step output — say, `issue 2`'s **Attribution** — pair the identifier with the step name; each issue keeps its own copy of every step.

Work the queue one issue at a time, running the full 20-step flow from a fresh **Explore** — never inherit another issue's output, even for the same file, and never pause between issues. **Read-through** and **Regression check** sometimes surface related issues; append each as its own fresh queue entry, run independently from its own **Explore**. Keep draining until nothing's left.

Once the queue is empty, report back to the caller. For each issue: the files it changed plus a one-line summary, or, for anything declined, the reason (`model-compliance` / `noise` / `non-issue`). List any unconnected observations separately.

---

## 20-Step Flow

**Expand check**, **Trim check**, and **KISS check** are the three checks in this flow; each produces a revised proposal in turn (**Expanded version**, **Trimmed version**, **KISS version**). Together they swing like a pendulum: **Expand check** pushes as far outward as the problem allows, **Trim check** swings back inward, **KISS check** settles on the first-principles balance between them. A swing counts as **narrow** — meaning **KISS version** ends up adding nothing beyond what **Initial proposal**'s own candidates already contained — when it's anchored to the prior step's wording (synonym-space drift) instead of freshly re-derived from **Problem**.

### Output structure (enforced)

Every step gets its own header line, `**Step N — <name>**` (e.g. `**Step 6 — Attribution**`), N matching its position in the numbered list below. Write that header *before* doing the step's work — never retrofit one around a summary after the fact. Folding two steps under one heading, covering multiple steps' revisions in one undifferentiated block of prose, or pointing back to an earlier step instead of writing this step's actual content (a parenthetical cross-link alongside real content is fine) — all of these count as skipping the step.

Every step from **Explore** through wherever the issue is currently being written or exited needs this header, with four carve-outs: a `model-compliance` exit needs headers only through **Attribution**; `noise` only through **Problem**; `non-issue` only through **Classify**; `trivial` needs them through **Initial proposal** (skipping **Research** before it), then jumps straight to **Apply**, skipping **Expand check** through **KISS version** after it. Every other case needs headers through **Initial proposal**, with **Research** included since it comes first.

Should **Problem**'s mission-relation conclusion shift mid-analysis — it looked like `noise`, but lands on `blocker`, `tension`, or `neutral` instead — the required range recalculates from that new conclusion. Each issue in a batch tracks this independently.

### The Iron Law

EVERY STEP RUNS IN FULL, NOT SKIPPED, NOT DONE SUPERFICIALLY. A STEP'S HEADER IS WRITTEN BEFORE ANY OF THAT STEP'S WORK. A STEP'S MANDATORY DISPATCH ACTUALLY HAPPENS, NOT JUST ITS SELF-PERFORMED EQUIVALENT. A STEP'S MANDATORY OUTCOME (e.g. **Review** finding a gap seeds a new Drain Queue entry) IS CARRIED OUT, NOT JUDGED UNNECESSARY. NO EXCEPTIONS.

A late header, a dispatch skipped or self-performed instead, a mandatory outcome waved off — none of these three can be fixed after the fact. The instant one happens it's already permanent in the transcript; a "redo" only adds more content after it, it doesn't undo it. Get it right the first time.

### Self-targeting

An edit to this SKILL.md file itself — any copy, wherever it lives — is this flow's issue input; run it through like any other target, no matter the framing (a raised issue, a copyedit, the agent's own unprompted initiative) and no matter the size ("too small to bother" doesn't excuse skipping a step). The flow's normal skip logic still applies here — `noise`, `model-compliance`, `non-issue`/`trivial` — and so does Drain Queue's per-issue scoping. What's ruled out is editing outside the flow, or delegating the edit and rubber-stamping the result — including delegating it to "this was already verified elsewhere." Every edit runs its own full flow.

### Dispatch discipline

Anything genuinely parallel and background-dispatched within this flow, don't circle back to poll or sleep-wait on it; the harness delivers the notification on its own.

A subagent receives exactly what its own step specifies, and never the dispatching agent's own expectation of how the result should come out. Where a step requires citing a fact, that citation has to name something a third party could independently check — a file:line, a transcript, a diff — not just the dispatching agent's own say-so; an uncheckable citation doesn't satisfy the requirement.

### The steps

1. **Explore** — before forming any conclusion, read what the issue and **Target location** point at: the flagged file or section, and whatever it visibly depends on. The tool calls themselves are this step's evidence; don't paraphrase them into a summary.
2. **Fact check** — list the concrete evidence already on hand for the issue as raised: file:line citations, existing artifact output, anything already on record. This step only states what's already pointable-at — building something new is **Red verify**'s job, deciding why is **Attribution**'s.
3. **Red verify** — proof, not restatement: where the issue has executable behavior, build a re-runnable reproduction and show it currently fails.
   - A code change: a real unit test or script, written and executed.
   - A prompt, instruction, or skill-spec change: dispatch an independent agent with real work, built around the identical, unparaphrased text the way the actor who'd actually encounter it naturally would — never a hypothetical scenario for the subagent to narrate, and never framed as a request to critique the wording. Let the problem surface on its own from genuine execution.
   - A genuine mix of both: pick whichever route yields a result you can cite and re-run.
   - What doesn't clear this step: a weak assertion, a test that never ran or was scoped to the wrong thing, or reasoning about what a run would probably show. Quote the composed dispatch prompt verbatim in this step's own output before calling `Agent`. **Green verify** re-runs this same artifact later — don't build a second one now.

   Judgment-based behavior still counts as executable here. "Pass" means a specific, named signal actually fired — a particular tool call, a file landing in a particular state — never "the agent judged it correct." The only substitute for a fresh reproduction is a prior real instance that's already fired and been observed, cited concretely — reachable-in-principle isn't enough. This extends to a dispatched agent's own tool-call compliance: judge it from the actual transcript; no transcript routes straight to **Attribution**'s `model-compliance` exit.

   Runs on the issue's own case alone.
4. **Context** — the cross-file, cross-reference background the fix needs. Every claim here MUST cite its source (file:line), and the target's stated purpose (from Inputs' **Context**) MUST be included with its own citation — later steps measure the fix against this purpose, and **Review** re-checks these citations once the real post-fix content exists.
5. **Problem** — confirm this is a real issue worth fixing. State its relation to the target's mission, citing the **Context** purpose plus the specific fact connecting them: `blocker` (serves the mission directly) / `noise` (doesn't align — consider dropping) / `tension` (trades off against the mission) / `neutral` (unrelated either way). A `noise` verdict exits the flow right here, carrying that same citation as its reason.
6. **Attribution** — replay the flagged instance through the target's current, unmodified logic — the reproduction **Red verify** built.
   - `source-variance` — the logic as written is correct; it only missed this instance because some bounded list or enumeration it relies on hadn't yet included this value. Cite that list (file:line). The logic itself is sound; what it checks has irreducible, ongoing variation no list will ever fully close. This still runs through **Classify** normally — it's not a `non-issue`.
   - `target-defect` — the logic has no branch or condition that would catch this input, regardless of its content. Cite the absence (file:line). A target with no content-dependent check at all resolves trivially here.
   - When the finding is about whether an actor complied with an instruction, rather than a static code gap, `target-defect` additionally requires that compliance be checkable by someone other than that actor — a tool-call transcript, a diff, a log, never self-attestation. Where no such evidence could ever exist, the verdict is `model-compliance` instead: exit here with the cited reason (the Drain Queue's final report already preserves that record — don't also mark the target file).
7. **Classify** — three stages, checked in order.

   **Stage 1 — `non-issue`**: already covered elsewhere. Exits the flow here.

   **Stage 2 — `trivial`**: only a typo, a rename, or a formatting fix qualifies as `trivial`. Skips **Research**, reaches **Initial proposal**, then skips **Expand check** through **KISS version** entirely, going straight from **Initial proposal** to **Apply**.

   **Stage 3 — default** (only when Stage 1 and Stage 2 don't apply): pick exactly one of ISO/IEC 14764's four software maintenance types, citing the fact it rests on and the nearest alternative type that fact rules out — a bare label with no ruled-out runner-up doesn't satisfy this step:
   - `corrective` (reactive + correction) — an already-manifested defect; fix it.
   - `preventive` (proactive + correction) — a latent defect that hasn't caused a failure yet; fix it before it does.
   - `adaptive` (proactive + enhancement) — the target needs to keep up with a changed external environment (a dependency, a convention, a related rule) it no longer matches.
   - `perfective` (reactive + enhancement) — not a defect; an improvement to performance, clarity, or maintainability based on feedback.

   This routes to the full divergent flow: three candidates at **Initial proposal**, **Expand check** widens further, **Trim check** fuses, **KISS check** gatekeeps.
8. **Research** — how do mature systems or the wider industry handle this class of issue? Cite at least one source; skip for `trivial`/`non-issue`. Don't let research override an established convention the target already adopted — cite the one it doesn't override.

   State a **search domain** before dispatching: one narrow technical topic, derived from both **Classify**'s class and **Problem**'s actual mechanism — the class alone is too generic. Treat it as a hard scope constraint.

   **MUST dispatch subagent.** Receives **Problem** (primary) + search domain + **Context** (reference). Prompt it to read discussion and critique beyond official docs, find structurally similar existing projects, check for an applicable formal model, gather evidence before sorting pro/con, and cite sources — dispatching up to 3 of its own subagents in parallel if the ground is disparate. Parent verifies the result against established principles before adopting, citing what it checked against.
9. **Initial proposal** — re-read **Problem**, then diverge into **three** candidate solutions that differ in fix approach and/or fix location, not merely in phrasing. Each candidate is written in prose, stating the fact it rests on. A `tension` issue also requires each candidate to cite the trade-off's cost.

    For **Stage 2** (`trivial`) issues: the three candidates are still produced, but Parent picks one directly, using its own judgment, and hands it straight to **Apply**.
10. **Expand check** — push the candidate-solution space as wide as possible: each lens produces additional candidate solutions the three from **Initial proposal** haven't tried, re-derived fresh from **Problem** itself — never from **Initial proposal**'s draft text.

    **MUST dispatch N≥3 parallel subagents**, each a genuinely distinct lens. One MUST be `mission-alignment` (receives **Context** as primary anchor + **Problem**, lists what a fix could serve or undermine). The rest are drawn from user-facing / robustness / observability / maintainability / edge-case / security, each receiving **Problem** as primary. Parent merges all lists into one, citing each lens's raw count against the final deduplicated count — a dropped candidate must be cited as a duplicate of a named surviving one, not silently absorbed.
11. **Expanded version** — the full candidate-solution list itself, written in prose: the three from **Initial proposal** plus whatever **Expand check** added.
12. **Trim check** — swing back inward: fuse the candidate pool into a single solution, keeping from each candidate only the part necessary to satisfy **Problem**.

    **MUST dispatch subagent to perform the fusion.** Receives **Problem** (primary) + **Expand check**'s list + **Expanded version** + **Context** (reference): deciding what's necessary from each candidate, fusing those parts into one coherent solution, resolving any contradiction between fused parts.
13. **Trimmed version** — the fused result, written in prose, citing which element of which candidate each part came from. This produces exactly one result.
14. **KISS check** — walk the 5 KISS rules below against **Trimmed version**, each in three distinct parts: **Hypothesized violation** (quote the specific clause, name a falsifiable violation) → **Verification** (cite the evidence either way; no evidence found counts as violated) → **Judgment** (violated or not, with the reason). Never collapse these three parts into a single line.

    **MUST dispatch adversarial subagent**, uninvested in **Trimmed version**. Receives **Problem** + the 5 rules + **Trimmed version** (the target) + **Expanded version** + **Context** (Rule 2 checks against it), prompted to find violations, not validate.
15. **KISS version** — **Trimmed version** plus whatever **KISS check** restored, derived from "the simplest change that actually satisfies **Problem**," written in prose. This goes into **Apply**.
16. **Apply** — `Edit` the change directly, narrating what changed in prose. Don't reach for `AskUserQuestion` just for approval to proceed — that's different from asking because the proposal itself has a real, unresolved gap.
17. **Green verify** — confirm every change in the applied change actually landed; re-apply anything that didn't. If **Red verify** built a reproduction, re-run that same artifact and confirm it passes, citing the actual output. Any supplementary case **Regression check** builds elsewhere follows this same red-then-green standard, checked against the real post-**Apply** file's own pre-fix and post-fix wording — not by reasoning about what it would probably show. A downstream issue discovered along the way doesn't get fixed here; it seeds its own fresh Drain Queue entry. Report each change applied (file:line + summary) and the reproduction's result.
18. **Review** — **MUST dispatch** an independent, uninvested subagent for three checks: whether the applied change actually resolves **Problem** — not merely whether **Green verify** turned green — whether **Context**'s own citations still hold up, and whether the applied change faithfully matches **KISS version**'s prose rather than drifting from it during the translation into a real edit. Receives **Problem** (primary), the real post-**Apply** file (not the diff), **Context**, and **KISS version**. Report its verdict, citing file:line in the actual content. Any of the three gaps it surfaces — **Problem** left unresolved, a **Context** citation that doesn't hold up, or a drift from **KISS version** — seeds a new Drain Queue entry; this step never reopens **Apply**.
19. **Read-through** — scan the target for problems similar to the one just fixed. State the full candidate list up front, before checking any of them — never assembled after the fact — then `Read` each one in full (`grep` only points at candidates, it never substitutes for reading them). Match the search method to the kind of edit made: added content → sections that look similar; removed content → `git diff` plus the first line of each affected section; a renamed identifier → every mention of it; a changed structure → sibling sections. Something only counts as related with a cited textual relationship (file:line) behind it — anything looser goes into the unconnected-observations list.

    **MUST dispatch one adversarial subagent, unconditionally — `trivial` included** (both of this rule's real-world failures happened at exactly this step). Receives **Problem**, the real post-**Apply** file, **Context**, **Target location** — not the parent's own candidate list, so it searches independently under the same discipline. Append whatever related problems it finds to the Drain Queue; don't fix them now.
20. **Regression check** — same discipline as **Read-through**: state candidates first, `Read` rather than `grep`, require a cited textual relationship, dispatch unconditionally. Apply it to regressions instead of related problems: re-read every section connected to the edit, across every file it touched. Cross-references → each section that mentions the changed thing; definitions → each section that uses it; schemas or interfaces → every producer and every consumer; rule lists → check for contradictions. Confirm nothing else broke — no dangling cross-reference, no contradiction, no new ambiguity. Append whatever regressions turn up to the Drain Queue; don't fix them now either.

### What "KISS" means here (for **KISS check**)

KISS = Keep It Simple, Stupid. Go back to first principles, keep only what's actually needed, stay true to the business facts, and hold up long-term. Applies to **KISS version** final; **Trimmed version** overshoot is a pendulum tool.

Five rules:

1. **Any simplification must first satisfy first principles** — return to essence
2. **No simplification may violate business logic** — business facts are not fat
3. **A good system must remain continuously explainable, verifiable, and maintainable**
4. **Real simplification cuts understanding, verification, and maintenance cost together**
5. **If a fix makes the future harder, it isn't KISS**
