# Expand `code-review` from two axes to three, with an adversarial-review → verification pipeline per axis

`code-review` shipped with two deliberately unmerged axes — **Standards** and **Spec** — on the argument that a change can pass one and fail the other, so blending them into one verdict lets one mask the other. That principle was never written down as its own decision; it lived only as a "Why two axes" section inside the skill itself. This ADR both supersedes the two-axis shape and retroactively records the reasoning it replaces.

The trigger was a request to fold in a third question — is the change well *designed*, not just standards-compliant and spec-faithful — and to harden each axis's findings with an independent verification pass before they're ever reported, using cost-efficient sub-agents for the fan-out.

## Decision

**Three axes, re-partitioned so they stay orthogonal — not three arbitrary labels over the same material.**

- **Features** (renamed from **Spec** — same mechanics, same issue-tracker sourcing, same "no spec found → skip and say so" fallback) — does the diff deliver what the issue/PRD asked?
- **Structure** (narrowed from the old **Standards**) — repo-documented conventions, plus only the *local/textual* half of the Fowler smell baseline (Mysterious Name, Duplicated Code, Data Clumps, Primitive Obsession, Repeated Switches).
- **Design** (new) — the *module-shape* half of the same baseline (Feature Envy, Shotgun Surgery, Divergent Change, Middle Man, Message Chains, Refused Bequest, Speculative Generality), re-expressed in the `codebase-design` skill's existing vocabulary (module, interface, depth, seam, leverage, locality) instead of being restated in new words.

The split matters because a naive three-way cut (Standards intact, Design bolted on) would have both Structure and Design flagging the same smells under different names — duplication, not orthogonality. Partitioning the baseline once, disjointly, is what keeps "a change can pass one axis and fail another" true for all three.

A **precedence rule** governs conflicts: a documented repo standard overrides a universal smell, and a universal smell overrides imported design vocabulary. Design is the lowest-authority axis — it imports a philosophy (`codebase-design`'s) the target repo may never have adopted, so every Design finding is a judgement call, never a hard rule the way a documented Structure standard can be.

**Pipeline: adversarial review, then verification, then inline assembly — not a longer chain.** Each axis runs one sub-agent that actively hunts for problems (reporting nothing when genuinely clean), then one independent sub-agent that adjudicates each raw finding as CONFIRMED or PLAUSIBLE before it's trusted. Two rejected alternatives:

- **A verifier per individual finding**, not per axis. Rejected: unbounded fan-out (3 axes × N findings) scales cost and latency in the number of findings on every routine diff — the wrong shape for a tool meant to run before every commit. One verifier per axis is a fresh, independent second look at bounded cost.
- **A dedicated synthesizer agent** to merge cross-axis findings. Rejected: dedup of "the same location flagged by more than one axis" is mechanical and the skill's own thread already holds every verified finding — spawning an agent for it added a lossy handoff and risked reintroducing the exact cross-axis blending the two-axis design was built to prevent. The three axes are still presented under separate headings; a shared location is cross-referenced under each, never merged into one verdict.

**No Opus, or any specific model, baked into the runtime pipeline.** The fan-out (3 review + 3 verify sub-agents) is pinned to a cost-efficient tier (`sonnet`/`haiku`, degrading to the inherited session model if unavailable) purely for cost control on a tool that runs on every diff. A higher-tier model was used once, out-of-band, to review *this decision* before it was built — that is a planning-time choice, not a property of the skill, and baking an expensive model into every third-party installer's routine `/code-review` run would be the wrong default.

**Built with the plain `Agent` tool, not `Workflow`.** This skill ships to arbitrary installers via the plugin. `Workflow` orchestration requires an explicit opt-in that a routine review tool shouldn't need to demand of every user just to run.

**Version:** landed as a `.changeset/three-axis-code-review.md` minor changeset, not a hand-edit of `package.json`. This repo versions through Changesets (`package.json`'s `version` and `CHANGELOG.md` are only updated by running `changeset version`, which consumes every pending changeset at once); a pending `ship-as-claude-plugin` changeset already sits alongside this one, which is why `package.json` (`1.1.0`) trails `.claude-plugin/plugin.json` (`1.2.0`) today — the plugin manifest was pre-set to the version that release will produce. That is expected Changesets behaviour, not drift, and neither manifest should be hand-edited outside an actual release. Treated as a minor bump: a behavior expansion of a Claude-facing skill, not a break in a programmatic API contract, even though the reported section headings move from `## Standards`/`## Spec` to `## Features`/`## Structure`/`## Design`.

## Invariants this creates

- The Fowler smell baseline is partitioned once, disjointly, between Structure (local/textual) and Design (module-shape) — a smell belongs to exactly one axis, never both.
- Every Design finding is phrased as a judgement call, never a hard violation — the precedence rule (repo standard > universal smell > imported design vocabulary) is checked whenever axes disagree.
- Sub-agent fan-out (review and verification) is pinned to a cost-efficient model tier, with graceful fallback to the inherited session model — no skill in this repo hard-fails on an unavailable model tier.
- The three axes are never blended into a single cross-axis verdict; a location flagged by more than one axis is cross-referenced under each heading instead.
- No runtime stage of `code-review` depends on a specific high-tier model — tier choice for the runtime fan-out is a cost lever, not a correctness dependency.
- Version bumps for this repo go through a changeset file (`.changeset/*.md`), never a hand-edit of `package.json`'s `version`; `.claude-plugin/plugin.json`'s `version` is only touched by whoever runs the actual release (`changeset version`), per `CLAUDE.md`/ADR-0002 — not by an individual feature change.
