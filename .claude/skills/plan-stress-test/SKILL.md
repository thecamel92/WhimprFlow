---
name: plan-stress-test
description: Adversarially stress-test a plan for TECHNICAL and ARCHITECTURAL flaws — decisions that would damage the long-term architecture or block where the product is heading. Usually pointed at a detailed implementation plan; sometimes at a vaguer one (a roadmap / rough direction). Use when the user invokes /plan-stress-test, or asks to "stress-test / poke holes in / red-team this plan", "what could go wrong architecturally", "find the structural problems with this plan". Read-only: it reports, it does not edit the plan or the code. Focuses on structural soundness (boundaries, contracts, data model, coupling, one-way doors, invariants); it does NOT judge whether the feature is worth building or whether the steps are in a nice order — unless getting that wrong would do severe, hard-to-undo architectural damage.
---

# Plan stress-test

Take a plan and try to break it — on **architecture**, not taste. The goal is to catch **structural decisions that will hurt later**: a boundary in the wrong place, a data shape that can't evolve, a shortcut that becomes load-bearing, an abstraction that won't survive the next milestone. Find the thing that is cheap to change **now** and ruinous to change **after it ships**.

This is the adversarial counterpart to designing a plan (`/brainstorm` shapes it, `/structure-plan` structures it). Here you attack the result.

## The one question behind every finding

**"When this plan's shape meets more scale, more cases, real persisted data, or the next milestone — what forces it to change, and how expensive is that change?"** A decision that's cheap to reverse is rarely worth flagging. A decision that calcifies — data you can't migrate, a contract every caller now depends on, a boundary that couples two things forever — is the whole point.

## What to hunt for (architectural failure modes)

- **Boundary drawn in the wrong place** — unrelated concerns fused into one unit, or one concern split across a seam so a single change touches both sides. Ask where the natural cut is and whether the plan cuts there.
- **Duplicated source of truth** — the same fact/shape defined in two places that must now be kept in sync by hand. Every future change has to remember both; eventually one drifts. Flag any "these two must always match" the plan creates.
- **Leaky or unstable contract** — the shape crossing a boundary exposes internal detail, or is defined loosely (so consumers guess), or will predictably need a breaking change once a second consumer/case appears.
- **One-way doors** — schema, persistence, id/key scheme, or public-contract decisions that are expensive or impossible to reverse **once real data or real callers exist**. Name the door and what it costs to walk back.
- **Load-bearing shortcut** — a "temporary" simplification that the rest of the system will come to depend on, so it can never actually be removed. The plan should say how it gets undone; if it can't, that's the finding.
- **Doesn't generalize to the stated direction** — the shape is correct for today's single case but contradicts or blocks where the project is explicitly heading. An abstraction pinned to "one" when the roadmap says it must become "many" (or vice-versa).
- **Contradicts an existing invariant** — the plan quietly violates a rule the codebase already relies on (a guarantee enforced in code, a documented architectural decision, a back-compat promise). Silent violations are the dangerous ones.
- **Trust / validation gap at a seam** — untrusted input (model output, user input, an external API, anything crossing a process boundary) consumed without a validating boundary, or an invariant assumed but never enforced where the data actually crosses.
- **Coupling that calcifies** — new dependency edges that make future change harder: a module reaching across a layer it shouldn't, hidden coupling via shared mutable/global state, a lower layer forced to know about a higher one.
- **Abstraction at the wrong level** — too rigid (bakes in an assumption that the plan's own future says must vary) or too generic (a framework built for exactly one caller).
- **Migration / back-compat hazard** — existing persisted data or existing callers the change breaks with no stated path for state that already exists.
- **Structurally baked-in scaling/perf trap** — an N+1, unbounded growth, a synchronous choke point, or a shape that won't hold at an order of magnitude more — when it's a consequence of the STRUCTURE, not a micro-optimization.
- **External contract treated as internal** — depending on a third-party API's shape/version as if it were owned and stable, instead of behind a boundary that can absorb its drift.

## What NOT to flag (scope)

Deliberately stay out of these — they are not this skill's job:

- Whether the feature is worth building, whether anyone wants it, product/market/priority judgment.
- The **order / sequencing** of the steps as a preference. (Exception below.)
- Naming, copy, formatting, and other bikeshedding; micro-optimizations with no structural consequence.

**The severity exception (both directions):** if a "product" or "ordering" choice would cause **severe, hard-to-undo architectural damage** — e.g. building a whole subsystem on a shape the long-term vision can't use, or an ordering that forces an irreversible decision before the information needed to make it exists — raise it. But frame it as its **architectural consequence**, and say explicitly that it's outside the usual scope so the user knows why you crossed the line.

## Calibrate to the plan's altitude

- **Detailed plan (the common case):** you have concrete contracts, types, schema, named producers/consumers. Scrutinize those directly — trace each boundary, check each shape against its consumers, look for the specific decision that bites. Findings should cite the exact contract/step.
- **Vague plan / roadmap:** there are no line-level contracts yet, so don't invent them. Attack the **directional bets**: does the proposed shape commit the architecture to something a later milestone will fight? What one-way doors does this direction imply? What contract will eventually be *forced*, and is it the right one? Say plainly that at this altitude the findings are about direction, not implementation detail.

## Method

0. **Consider delegating the attack to a fresh context.** If you helped *write* the plan you're now attacking, you are anchored to your own reasoning and will defend the premises you built — the deepest flaws are invisible from inside. Send the **`plan-critic`** agent the plan text **and not the conversation**: never having seen the rationale is its entire advantage, so it can ask "why this shape at all?" without owning the answer. Then judge its findings here. Skip this only when the plan came from elsewhere and you're already reading it cold. `plan-critic` is this skill's arm — it is not a separate entry point, so nothing is lost by always routing through here.

1. **Extract the plan's spine.** State, in your own words, the core change and — most importantly — its **boundaries and contracts**: what data crosses where, who produces and who consumes each shape. If the plan doesn't make its boundaries explicit, infer them — *a plan that hides its boundaries is itself a finding.*
2. **Anchor to reality — don't theorize.** Read enough of the **actual codebase** to ground each claim, and check the project's stated **long-term direction and invariants** (the architecture record and top-level project instructions are where those live). A finding must be about this system as it really is and is meant to become, not a generic worry.
3. **Push each decision forward in time.** For every boundary/shape/default, run the one question above: 10× scale, the next milestone, real persisted data, a second consumer. Find where it breaks and price the fix.
4. **Rank by (long-term blast radius × irreversibility).** A small, easily-reversed wart ranks below a load-bearing one-way door even if the wart is more certain.

## Grounding bar for a finding

Every finding must name: **(a)** the exact decision (cite the plan step or the file/contract it touches), **(b)** a **concrete** future scenario where it fails — real inputs/state, not "this might not scale", **(c)** why it's hard to undo once shipped, and **(d)** a better shape or a mitigation. If you can't produce (b) and (c), it's a nitpick — cut it.

## Output

- **Verdict first (one line):** architecturally sound / sound with caveats / has a load-bearing flaw.
- **Ranked findings, most damaging first.** Each: **Decision** · **Long-term risk** · **Failure scenario** (concrete) · **Reversibility** (how stuck are we once it ships) · **Suggested shape** · a confidence tag (**confirmed** vs **plausible — depends on X**).
- **Solid foundations:** a short note on what the plan gets architecturally *right* — so it's clear you actually examined it and the user doesn't over-correct good decisions.
- **Out of scope (not judged):** explicitly list what you deliberately didn't assess (feature worth, step ordering, style), so your silence there isn't mistaken for endorsement.
- **If nothing serious survives scrutiny, say so plainly.** Do not manufacture issues to look thorough — a clean bill on a sound plan is a valid, valuable result.
- **One closing improvement — but only if it clears the bar.** End with a single, most-impactful improvement the plan should adopt, **only when a genuinely impactful one exists**. The bar is high: it must materially strengthen the architecture or unblock the long-term direction. If the plan is ~99% sound, the last 1% does **not** count — do NOT suggest fewer lines of code, a tidier refactor, a nicer name, or any change whose absence costs nothing real. When there is no improvement that meets this bar, say "no material improvement to add" and stop. One real lever beats a list of polish.

## Tone

Adversarial toward the architecture, never the author. Prefer surfacing **one real one-way door** over ten cosmetic nitpicks. When you're unsure something is a genuine problem, say so and name the condition under which it bites — an honest "this is risky *only if* Y" is more useful than false confidence. Explain each finding in plain language and define any jargon as you go, so a non-specialist can follow *why* the decision is dangerous, not just that it is.
