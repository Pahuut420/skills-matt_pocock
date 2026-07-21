---
name: code-review
description: Review the changes since a fixed point (commit, branch, tag, or merge-base) along three axes — Features (does it match the issue/PRD?), Structure (repo conventions and code smells), and Design (sound module shape — deep interfaces, clean seams). Each axis runs an adversarial review then a verification pass. Use when the user wants to review a branch, a PR, work-in-progress changes, or asks to "review since X".
---

Three-axis review of the diff between `HEAD` and a fixed point the user supplies. Each axis runs its own **adversarial review → verification** pair of parallel sub-agents, so none pollute each other's context:

- **Features** — does the code faithfully implement the originating issue / PRD / spec?
- **Structure** — does the code conform to this repo's documented coding standards, and avoid textual code smells?
- **Design** — is the module shape sound: deep interfaces, clean seams, no misplaced responsibility?

Findings are never blended across axes — see _Why three axes_ — but a location flagged by more than one axis is cross-referenced under each.

The issue tracker should have been provided to you — run `/setup-matt-pocock-skills` if `docs/agents/issue-tracker.md` is missing.

## Process

### 1. Pin the fixed point

Whatever the user said is the fixed point — a commit SHA, branch name, tag, `main`, `HEAD~5`, etc. If they didn't specify one, ask for it.

Capture the diff command once: `git diff <fixed-point>...HEAD` (three-dot, so the comparison is against the merge-base). Also note the list of commits via `git log <fixed-point>..HEAD --oneline`.

Before going further, confirm the fixed point resolves (`git rev-parse <fixed-point>`) and the diff is non-empty. A bad ref or empty diff should fail here — not inside three parallel sub-agents.

### 2. Identify the Features source

Look for the originating spec, in this order:

1. Issue references in the commit messages (`#123`, `Closes #45`, GitLab `!67`, etc.) — fetch via the workflow in `docs/agents/issue-tracker.md`.
2. A path the user passed as an argument.
3. A PRD/spec file under `docs/`, `specs/`, or `.scratch/` matching the branch name or feature.
4. If nothing is found, ask the user where the spec is. If they say there isn't one, the **Features** sub-agent will skip and report "no spec available".

### 3. Identify the Structure sources

Anything in the repo that documents how code should be written, such as `CODING_STANDARDS.md` or `CONTRIBUTING.md`.

On top of whatever the repo documents, the Structure axis always carries the **local smell baseline** below — a fixed set of Fowler code smells (_Refactoring_, ch.3) confined to a single hunk or file, that applies even when a repo documents nothing. Two rules bind it, and also govern the Design axis's baseline in step 4:

- **The repo overrides.** A documented repo standard always wins; where it endorses something the baseline would flag, suppress the smell.
- **Always a judgement call.** Each smell is a labelled heuristic ("possible Duplicated Code"), never a hard violation — and, like any standard here, skip anything tooling already enforces.

Local smell baseline — *what it is* → *how to fix*:

- **Mysterious Name** — a function, variable, or type whose name doesn't reveal what it does or holds. → rename it; if no honest name comes, the design's murky.
- **Duplicated Code** — the same logic shape appears in more than one hunk or file in the change. → extract the shared shape, call it from both.
- **Data Clumps** — the same few fields or params keep travelling together (a type wanting to be born). → bundle them into one type, pass that.
- **Primitive Obsession** — a primitive or string standing in for a domain concept that deserves its own type. → give the concept its own small type.
- **Repeated Switches** — the same `switch`/`if`-cascade on the same type recurs across the change. → replace with polymorphism, or one map both sites share.

### 4. Identify the Design vocabulary and read beyond the diff

The Design axis speaks the **module** vocabulary from `codebase-design` — reuse these terms exactly:

- **Module** / **Interface** — anything with an implementation, and everything a caller must know to use it correctly.
- **Depth** — leverage at the interface: **deep** is a lot of behaviour behind a small interface; **shallow** is an interface nearly as complex as what's behind it.
- **Seam** — the place a module's interface lives, where behaviour can vary without editing there. One adapter at a seam is a hypothetical seam; two is a real one.
- **Leverage** / **Locality** — what depth buys callers (leverage) and maintainers (locality): fix once, fixed everywhere.

Module-shape baseline — the other half of the Fowler smells, re-expressed in this vocabulary:

- **Feature Envy** (locality) — a method that reaches into another module's data more than its own. → move the method onto the module it envies.
- **Shotgun Surgery** (locality) — one logical change forces scattered edits across many files in the diff — no seam absorbs it. → gather what changes together into one module.
- **Divergent Change** (seam placement) — one module is edited for several unrelated reasons — the seam is in the wrong place. → split so each module changes for one reason.
- **Middle Man** (shallow module) — a module that mostly just delegates onward, its interface about as complex as its implementation. → cut it, call the real target direct.
- **Message Chains** (leaky interface) — long `a.b().c().d()` navigation the caller shouldn't have to know. → hide the walk behind one method on the first module.
- **Refused Bequest** (wrong adapter) — a subclass or implementer that ignores or overrides most of what it inherits — it doesn't actually fit the seam. → drop the inheritance, use composition.
- **Speculative Generality** (hypothetical seam) — abstraction, parameters, or hooks added for needs the spec doesn't have — an adapter for a seam nothing varies across yet. → delete it; inline back until a real need shows.

A diff shows a hunk, but seam and depth judgements need the module around it — the Design sub-agent (step 5) is explicitly licensed to read the surrounding file(s), not just the changed lines.

### 5. Spawn the three adversarial-review sub-agents in parallel

Send a single message with three `Agent` tool calls, one per axis, pinned to a cost-efficient model tier (`sonnet` or `haiku`) — fall back to the session's inherited model if that tier isn't available, don't hard-fail. Use the `general-purpose` subagent for all three.

**Features sub-agent prompt** — include:

- The diff command and commit list.
- The path or fetched contents of the spec.
- The brief: "Report: (a) requirements the spec asked for that are missing or partial; (b) behaviour in the diff that wasn't asked for (scope creep); (c) requirements that look implemented but where the implementation looks wrong. Quote the spec line for each finding. Report nothing if the diff genuinely delivers the spec as asked — don't pad. Under 400 words."

If the spec is missing, skip this sub-agent and note it in the final report.

**Structure sub-agent prompt** — include:

- The full diff command and commit list.
- The standards-source files found in step 3, **plus the local smell baseline from step 3** pasted in full — the sub-agent has no other access to it.
- The brief: "Report — per file/hunk where relevant — (a) every place the diff violates a documented standard: cite the standard (file + rule); and (b) any local smell you spot: name it and quote the hunk. Distinguish hard violations (documented-standard breaches) from judgement calls (smells, always judgement calls) — a documented repo standard overrides the baseline. Skip anything tooling enforces. Report nothing if the diff is genuinely clean on this axis — don't pad. Under 400 words."

**Design sub-agent prompt** — include:

- The full diff command and commit list, plus explicit permission to read the surrounding file(s) around each hunk.
- The module vocabulary and module-shape baseline from step 4, pasted in full.
- The brief: "Report — per module touched — any module-shape smell you spot: name it, quote the hunk, and say which term it violates (depth, seam placement, locality, leverage). Always a judgement call — this axis imports a design philosophy the repo may not have adopted, so frame every finding as 'possible X', never a hard rule; a documented repo standard, if one conflicts, always overrides it. Report nothing if the diff is genuinely clean on this axis — don't pad. Under 400 words."

### 6. Spawn the three verification sub-agents in parallel

Send a single message with one `Agent` tool call per axis that produced findings in step 5 (skip an axis with none) — same model tier as step 5. Each verifier is a fresh agent, independent of the one that authored the findings it's checking.

Give each verifier the diff and that axis's raw findings from step 5. Brief: "For each finding, independently check whether it holds up against the diff. Label each **CONFIRMED** (holds up) or **PLAUSIBLE** (arguable but not certain) — drop anything that doesn't survive scrutiny at all. Under 300 words."

### 7. Assemble

Present the verified findings under `## Features`, `## Structure`, and `## Design` headings, per axis, each tagged CONFIRMED or PLAUSIBLE. When the same file/hunk location is flagged by more than one axis, cross-reference it under each heading rather than merging into one combined verdict — the three axes stay separate all the way through (see _Why three axes_).

End with a one-line summary: total findings per axis, and the worst issue _within each axis_ (if any). Don't pick a single winner across axes — that's the reranking the separation exists to prevent.

## Why three axes

A change can pass one axis and fail the others:

- Code that follows every convention but implements the wrong thing → **Structure pass, Features fail.**
- Code that does exactly what the issue asked but breaks the project's conventions → **Features pass, Structure fail.**
- Code that's clean and correct but built on a shallow module or a seam in the wrong place → **Structure and Features pass, Design fail.**

Reporting them separately stops one axis from masking another. The three carry different authority, though — when they conflict: **a documented repo standard overrides a universal smell, and a universal smell overrides imported design vocabulary.** Design is the lowest-authority axis — it imports the `codebase-design` philosophy, which the target repo may never have adopted, so every Design finding is a judgement call, never a hard rule the way a documented Structure standard can be.
