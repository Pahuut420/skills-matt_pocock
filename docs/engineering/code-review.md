Quickstart:

```bash
npx skills add mattpocock/skills --skill=code-review
```

```bash
npx skills update code-review
```

[Source](https://github.com/mattpocock/skills/tree/main/skills/engineering/code-review)

## What it does

`code-review` reviews the diff between `HEAD` and a fixed point you supply — a commit, branch, tag, or merge-base — along three separate axes: **Features** (does it implement what the originating issue or spec asked for?), **Structure** (does the code follow this repo's documented conventions and avoid textual code smells?), and **Design** (is the module shape sound — deep interfaces, clean seams, no misplaced responsibility?). Each axis runs its own adversarial-review sub-agent, then an independent verification sub-agent that confirms or discards its findings, and the three are reported side by side. It never blends or re-ranks findings across axes — keeping them separate is the whole point, because a change can pass any one axis and fail the others, and a single blended verdict lets one mask another.

## When to reach for it

Type `/code-review`, or the agent reaches for it automatically when you ask to review a branch, a PR, work-in-progress changes, or anything "since X".

Reach for this when there is a diff to judge against a known-good point and you want three questions — *is it the right thing?*, *is it written the way this repo writes code?*, and *is it shaped well?* — answered independently. It runs at the end of the build loop; for actually writing the code test-first, use [tdd](https://aihero.dev/skills-tdd), and for building a whole spec into code use [implement](https://aihero.dev/skills-implement), which runs its own `/code-review` pass before committing.

## Prerequisites

The **Features** axis needs somewhere to find the originating spec — an issue reference in the commit messages, a path you pass in, or a spec under `docs/`/`specs/`. That issue-tracker wiring comes from [setup-matt-pocock-skills](https://aihero.dev/skills-setup-matt-pocock-skills); without a spec the Features axis simply skips and says so. The **Structure** axis needs nothing set up — it always carries a built-in Fowler smell baseline even in a repo that documents no conventions. The **Design** axis needs nothing set up either, but it reads a little past the diff — the surrounding file(s) around each hunk — since seam and depth judgements can't be made from a hunk alone.

## Three axes, never merged

The defining idea is the **three axes**, and the pipeline each runs through: adversarial review, then verification. **Features** asks the question of intent — does the code do what the issue or spec actually asked, without missing requirements or smuggling in scope creep? **Structure** asks whether the diff conforms to how this repo writes code — its `CODING_STANDARDS.md` or `CONTRIBUTING.md`, plus a fixed baseline of textual Fowler smells (Mysterious Name, Duplicated Code, Data Clumps, …). **Design** asks whether the module shape itself is sound, in the `codebase-design` vocabulary — deep interfaces, clean seams, no misplaced responsibility — using the other half of the Fowler smells (Feature Envy, Shotgun Surgery, Middle Man, …) read as module-shape problems rather than restated as text-level ones.

Each axis first runs an adversarial sub-agent that hunts for real problems, then a second, independent sub-agent that verifies each raw finding and labels it CONFIRMED or PLAUSIBLE before it's ever reported — a finding has already survived a skeptical second look by the time you see it. The final report presents all three under separate `## Features`, `## Structure`, and `## Design` headings with a per-axis summary; a location flagged by more than one axis is cross-referenced under each rather than merged into a single verdict. When axes disagree, a documented repo standard always overrides a universal smell, and a universal smell always overrides the imported Design vocabulary — Design findings are the softest, always framed as judgement calls.

## It's working if

- It pins and confirms the fixed point first (`git rev-parse`), failing fast on a bad ref or empty diff rather than inside the sub-agents.
- Features, Structure, and Design findings arrive in three distinct blocks, each citing its source — a quoted spec line, a repo standard or smell, or a named module-shape term.
- Every reported finding carries a CONFIRMED or PLAUSIBLE tag from the verification pass.
- When no spec can be found, the Features axis reports "no spec available" instead of inventing requirements.

## Where it fits

`code-review` is the review step at the tail of the main build chain:

```txt
grill-with-docs → to-spec → to-tickets → implement → code-review
```

Its closest neighbour is [implement](https://aihero.dev/skills-implement), which drives the build and calls this as its own review pass before committing; upstream, the spec it checks against is produced by [to-spec](https://aihero.dev/skills-to-spec) and [to-tickets](https://aihero.dev/skills-to-tickets). When you're unsure which skill or flow fits, [ask-matt](https://aihero.dev/skills-ask-matt) routes you.
