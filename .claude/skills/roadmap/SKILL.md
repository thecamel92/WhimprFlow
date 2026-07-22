---
name: roadmap
description: LONG-TERM planning altitude — decides WHAT to build next and WHY NOW from a vision that has many possible directions, then sequences it into ordered, numbered milestones with the critical ones flagged. Does NOT touch code and does NOT design technical structure. Proposes a target + reasoning, WAITS for your acceptance, and only THEN persists anything. Use when the user invokes /roadmap, pastes a vision/product doc with many directions, or asks "what should I build next", "what's the right order", "is now the time for X", "sequence these features", "break this big thing into milestones". Chain-with-fallback: hands a chosen target down to /brainstorm. Distinct from /brainstorm (which shapes ONE already-chosen target technically).
---

# roadmap

The **long-term altitude** — the top floor, above all the technical skills. It answers one question and only one: **what should I build next, and why now?** It ingests a vision with many possible directions (a product doc, a feature wishlist, a "here's everything I eventually want" brain-dump) and turns it into an **ordered, numbered sequence** with the load-bearing steps flagged.

It **never touches code** and it **never designs the technical structure** of anything — that is `/brainstorm`'s job one floor down. A long-term plan is not a technical plan; keeping them apart is the whole reason this skill exists as its own thing.

> **Jargon, once:** a *"one-way door"* is a decision that is expensive or impossible to undo later (a data model everything comes to depend on, a platform you build your whole product on top of). A *"two-way door"* is cheap to reverse — you can just try it and back out. A *"dependency"* here means one thing that must exist *before* another can work (you can't test a conversion-optimising feature before the thing it optimises exists).

---

## The one failure mode this skill exists to prevent

**Building the right thing at the wrong time.** Sequencing by *excitement* instead of by *dependency and reversibility* — sinking weeks into the shiny moat feature before the core loop it depends on even exists, or walking through a one-way door you never noticed was one-way.

**The governing rule of this skill:**

> **Order by what depends on what, and flag what you can't undo — before you order by what you're excited about.** A roadmap that lists features in the order you thought of them is a wishlist. A roadmap earns the name only when each item's *position* is justified by (a) what must exist first, and (b) whether getting it wrong is cheap or expensive to reverse.

---

## How to run it

1. **Take in the whole vision, then find the directions.** Read the doc / brain-dump the user brings. Pull out the distinct *directions* — the separable things that could each be built. Don't yet judge them; just name them.
2. **Score each direction on the three things that actually set order** (not on how cool it is):
   - **Depends on** — what must already exist for this to be worth building? (A feature that measures the core loop is worthless before the loop exists.)
   - **Unblocks** — what does finishing this make possible next?
   - **Reversibility** — is this a two-way door (try it, back out cheap) or a one-way door (a data model / platform bet everything downstream will lean on)? **Flag every one-way door.**
   - *(Optional, for a product with a competitive angle:)* does this **defend the core bet**, or does it **drift onto commodity turf** anyone could build?
3. **Sequence into a numbered list, foundation-first.** Order so that nothing appears before its dependencies. Number the steps. **Flag the critical few** — the one-way doors and the foundations everything else waits on — visually (e.g. `★ CRITICAL`), because those are the ones worth slowing down for.
4. **Name the single next target + why now.** Out of the sequence, call out the *one* thing to do next and the one-paragraph reason it's now and not later. That target is what you hand down to `/brainstorm`.
5. **Propose — then STOP and wait.** Present the sequence and the next target as a proposal. **Do not persist anything yet.** Ask the user to accept, re-order, or push back.
6. **Only on acceptance, persist.** Once the user says "yes, this is the plan," *then* write it to a durable doc — matching the repo's convention (a roadmap / unfinished-work / planning doc; ask once if placement is ambiguous). Until acceptance, nothing is written to disk. This keeps the skill from littering any repo with speculative plans.

---

## Chain-with-fallback

- **Chained:** the accepted **next target** becomes the opening line of `/brainstorm` ("here's the one thing we're shaping").
- **Alone:** run it any time you're staring at a pile of directions and don't know the order. It works standalone — it just needs the vision as input, which you supply.
- **Where it sits in the 55-min → one-prompt workflow:** roadmap picks the target *before* the technical trio (`brainstorm → structure-plan → dataflow-verifier`) shapes, builds, and proves it. Its output is not part of the coding prompt — it's what *chooses* which coding prompt you're about to write.

---

## What the output looks like

- **The numbered sequence** — each item one line, with its `depends on` / `unblocks` / reversibility noted where it matters, and the critical few flagged.
- **The next target** — the single thing to do next, with the "why now" paragraph.
- **The rejected orderings** — one line on any sequence you considered and dropped (e.g. "moat-first — rejected: nothing to measure yet"), so the order reads as *chosen*, not *default*.

---

## Quick self-check

- Did every item's **position** trace back to a dependency or a reversibility call — not to how exciting it is?
- Did you **flag the one-way doors** and the foundations explicitly?
- Is there a **single named next target** with a real "why now" — not a vague "do all of it"?
- Did you **wait for acceptance before writing anything** to disk?
- Did you stay at the **long-term altitude** — no code, no technical structure (that's `/brainstorm`)?
