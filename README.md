[![license](https://img.shields.io/github/license/miku3920/migakiage-skills)](https://github.com/miku3920/migakiage-skills/blob/main/LICENSE)

# migakiage-skills

A collection of skills for hardening and refinement.

## Setup

```bash
npx skills add miku3920/migakiage-skills
```

## Table of Contents

- [Skills](#skills)
- [Usage](#usage)
- [FAQ](#faq)
- [Contact](#contact)

## Skills

| Skill | Description |
|---|---|
| [`finding-resolution`](skills/finding-resolution/SKILL.md) | Drains a batch of reviewer-raised issues (code review, spec review, bug reports, etc.) through a rigorous 20-step fix flow. |
| [`skill-optimizer`](skills/skill-optimizer/SKILL.md) | Iteratively optimizes an existing skill by running N parallel test runs against it, fixing real issues via `finding-resolution` until convergence. |

`skill-optimizer` invokes `finding-resolution` mid-run (via the `Skill` tool) to apply its fixes — the two are not independent of each other. See each skill's own `SKILL.md` for full triggering conditions and behavior.

## Usage

Skills are automatically available once installed. The agent uses them when relevant tasks are detected.

```
Iterate on my code-review skill against these three test repos until it stops finding new issues
```

```
The reviewer flagged 4 issues in this PR — fix them properly, not just patch and move on
```

## FAQ

**1. Why does `skill-optimizer` need `finding-resolution` installed too?**

`skill-optimizer`'s outer loop hands every real issue it finds to `finding-resolution` for the actual fix, instead of keeping its own copy of the 20-step flow. `npx skills` does not install dependencies automatically, so if you only pick `skill-optimizer` during setup, it will fail partway through a run.

**2. Can I use `finding-resolution` without `skill-optimizer`?**

Yes. `finding-resolution` is a standalone skill for draining any batch of reviewer-raised issues (code review, spec review, bug reports) — it doesn't require `skill-optimizer` at all.

## Contact

Created by [@miku3920](https://github.com/miku3920). Feel free to open an issue if you have any questions or run into any problems.
