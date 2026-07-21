---
"mattpocock-skills": minor
---

Expand **`code-review`** from a two-axis review (Standards + Spec) to three: **Features** (renamed from Spec — does it implement what the issue/PRD asked?), **Structure** (repo conventions plus the local/textual half of the Fowler smell baseline), and **Design** (the module-shape half of the same baseline — Feature Envy, Shotgun Surgery, Middle Man, and friends — re-expressed in the `codebase-design` skill's vocabulary of module, interface, depth, seam, leverage, and locality). Splitting the old Standards baseline this way keeps all three axes orthogonal instead of Structure and Design double-flagging the same smells.

Each axis now runs an **adversarial review** sub-agent (hunts for real problems, reports nothing when genuinely clean) followed by an independent **verification** sub-agent that labels every finding CONFIRMED or PLAUSIBLE before it's reported — one verifier per axis, not one per finding, to keep cost and latency bounded. The three axes are still never blended into a single verdict; a location flagged by more than one axis is cross-referenced under each heading instead. When axes disagree, a documented repo standard overrides a universal smell, and a universal smell overrides the imported Design vocabulary — Design findings are always the softest, framed as judgement calls.

The review and verification fan-out is pinned to a cost-efficient model tier (falling back to the inherited session model if unavailable); no stage depends on a specific high-tier model. See `.agents/adr/0003-three-axis-code-review.md` for the full reasoning, including why this stayed on the plain `Agent` tool rather than `Workflow`.
