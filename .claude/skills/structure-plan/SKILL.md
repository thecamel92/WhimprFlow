---
name: structure-plan
description: TECHNICAL altitude, the workhorse — turns a chosen shape into a build a coding agent can execute in ONE shot AND a human can actually judge. Opens by framing the core issue + core solution in plain English, then two mandatory moves — (1) define the CONTRACTS — for every boundary, the explicit data shape that crosses it, ONE authoritative definition every stage refers to, with who-produces / who-consumes noted; (2) sequence into foundation-first, independently-shippable steps, each carrying its own check inline — and pairs every contract and step with an easy-to-understand plain-English explanation alongside the technical shape (not types alone). Language-agnostic — Pydantic, TypeScript types, a JSON schema, a plain struct, a docstring header — it enforces the MOVE, not the syntax. Use ONLY when the user explicitly invokes /structure-plan. This is a niche, deliberately-invoked skill — do NOT fire it for plan mode, for a generic "plan this", or for any other planning request. Chain-with-fallback: takes a shape from /brainstorm, hands each step's check to /dataflow-verifier.
---

# structure-plan

The **workhorse** of the technical trio, and the one that most directly feeds the single big Claude Code prompt you fire after planning. `/brainstorm` chose the *shape* and named the *boundaries*; `/structure-plan` turns that into a build an agent can execute in one shot — by making the two structural moves you can't yet make on reflex.

Those two moves are the entire skill: **define the contracts, then sequence foundation-first.** Everything else is detail.

> **Jargon, once:** a *"contract"* (also *"wire"*) is the explicit, written-down shape of the data that crosses one boundary — one authoritative definition that every stage on both sides refers to, instead of each stage guessing. It is *the backbone*: when something breaks at 2am, the contract is the one place you look to see what *should* have crossed. *"Foundation-first"* means the step that defines those contracts comes before any step that uses them.

---

## The one failure mode this skill exists to prevent

**Leaving the boundary shapes implicit — so stages pass each other bare dicts / tuples / loose objects, and a wrong shape slides four stages downstream before it surfaces as a weird symptom far from its cause.** For a self-taught debugger this is the expensive failure: there's no *one place to look*, because the shape was never written down.

**The governing rule of this skill:**

> **Every boundary `/brainstorm` named becomes ONE explicit contract, defined once and referred to everywhere — and the contract step is always step one.** A plan that jumps to feature steps without first pinning the shapes that cross its seams is a wishlist with numbers on it. Pin the shapes first; then the steps have a backbone to hang on.

---

## Move 0 — Frame it before you shape it (context + solution, in plain English)

Before any contracts, open the plan with two short paragraphs. This is the *why* the rest of the plan hangs on, and it's what lets someone who isn't already deep in the code actually evaluate the plan instead of nodding along to a wall of types:

- **The core issue (brief).** In 2–4 plain sentences: what problem is this build solving, and *why does it need solving*? Name the concrete pain — what breaks, what's missing, what's awkward today — **not** the solution yet. A reader should grasp what's at stake without knowing the codebase.
- **The core solution (brief).** In 2–4 plain sentences: the *shape* of the fix — the one idea the whole plan turns on (the shape `/brainstorm` chose) — in everyday language. Give the intuition, not the file list or the types. *"We give products a stable name-tag so every part of the app can point at the same one"* beats *"introduce a `Product[]` registry keyed by id."*

Keep both brief — a few sentences each. They set the frame; the contracts and steps fill it in. If you can't state the issue and the solution simply, you don't understand the change well enough to plan it yet.

---

## Move 1 — Define the contracts (the backbone)

For **every boundary `/brainstorm` named**, write the explicit shape of the data that crosses it. This is the generalised, mandatory version of the `contracts.py` insight: *one definition, reused everywhere.*

> **A contract for an EXISTING seam must be read, never recalled.** If the seam already exists, the real shape — including whoever quietly re-declares it — is a fact to go and get: send the **`dataflow-verifier`** agent in AUDIT mode. Before removing or changing an existing shape, send a **`blast-radius`** agent; it walks build-time / runtime / cross-process-contract / already-persisted data and finds the consumers no grep sees (a key inside an on-disk settings/state file written by an older version, a field frozen into a wire format). A plan that misses those is a plan that gets paid for twice. Both return an answer, not files.

For each contract, record:
- **A plain-English twin (REQUIRED — the qualitative half).** Alongside the technical shape, write 1–2 everyday-language sentences: what this data *is* in real-world terms, and why this boundary matters. The shape tells a coding agent what to build; the twin tells a human what it's *for*. Define any jargon on first use. **A contract that is only a type definition has done half the job** — pair every one with its twin.
- **The shape** — the fields and their types, in whatever form the codebase uses. **Language-agnostic:** Pydantic models, TypeScript `type`/`interface`, a JSON schema, a plain struct, even a documented dict at the top of a module. The skill enforces the *move* — an explicit, single, shared definition — **not** the syntax. Match the repo.
- **Who produces it and who consumes it** — name both ends of the pipe. A contract with no named producer/consumer isn't a boundary, it's a guess.
- **The wire-vs-internal split** — only what *crosses* a boundary is a contract. A stage's *private internal computation* (a scratch object it builds and never hands on) stays **local** to that stage and does *not* go in the shared definition. Keeping internal dialects from leaking across seams is half the value.
- **Load-bearing distinctions worth stating** — e.g. a three-state result where *"couldn't determine"* must never collapse into *"failed"*; a field that's optional-until-a-later-stage-fills-it. These are exactly the things a naive wiring gets wrong, so write them into the contract as notes.

This is the highest-leverage section of the whole plan. Spend words here. When you later run `/dataflow-verifier`, these contracts become what its VERIFY mode checks — the boundaries → contracts → assertions thread closes here.

---

## Move 2 — Sequence into foundation-first, shippable steps

Break the build into an **ordered list of steps** where:
- **Each step carries a plain-English twin (REQUIRED).** Pair the technical description (files/seams/change) with 1–2 plain sentences: what this step *accomplishes*, what will look or behave differently after it, and why it sits *here* in the order. The technical line is for the agent that builds it; the twin is for the human deciding whether the plan is right.
- **Step one is always the contracts / data model** (Move 1). Everything downstream imports it, so it lands first — often with *"no visible change yet"* as its honest success criterion.
- **Each step is sized for one focused effort** and **leaves the thing working** — shippable on its own, back-compatible where it touches existing behaviour. This is what turns a 50-item mush into an ordered spine the one big prompt can actually execute.
- **Each later step depends only on earlier ones.** No forward references.
- Name the **files/seams** each step touches (grounded in the real code — no invented paths). Group a repeated pattern once rather than listing identical work.

---

## Every step carries its check inline

A step without a way to prove it worked **isn't a step.** For each step, attach the check that confirms it — the lightest one that would actually catch a break at that step (a type-check, a unit test, a run-it-and-look, an assertion at the boundary the step just built). This is the handoff to `/dataflow-verifier`, which turns these inline checks into an actual verified boundary. Don't defer verification to the end; bake it into each step.

## Recordkeeping — plan the notes, don't just plan the code

For a build with any real weight, say up front **what durable notes get updated and when** — the design decisions and their *why*, so the reasoning survives after the code is written. Keep it general: a decision log, the repo's design docs, a note in whatever architecture record the project keeps. Name the specific file if the repo has one; otherwise flag that the record needs a home. (Per the persistence rule, actually *writing* durable docs waits until the plan is accepted.)

---

## Chain-with-fallback

- **Chained:** takes the boundaries + chosen shape from `/brainstorm` and turns them into contracts + steps. Its output — **contracts up top, ordered steps each with its check** — is the *body* of the one big Claude Code prompt.
- **Alone:** if the shape is already obvious, run `/structure-plan` directly; it just asks you for the shape and the boundaries that `/brainstorm` would have supplied, instead of breaking.

---

## What the output looks like — the build plan

- **The framing (Move 0)** — the core issue + the core solution, briefly, in plain English, up top.
- **Contracts** — one authoritative shape per boundary, with producer/consumer, the wire-vs-internal split, **and a plain-English twin**.
- **Ordered steps** — foundation (contracts) first, each shippable, each naming its files/seams, **each with a plain-English twin**.
- **Each step's inline check** — the proof it worked, handed to `/dataflow-verifier`.
- **Recordkeeping note** — what gets recorded and where (written only once accepted).

The rule of thumb: **every technical statement in the plan has a human-readable companion.** The contracts and steps are the skeleton; the framing and the twins are what make it a plan a person can actually judge — not just a spec an agent can execute.

---

## Quick self-check

- Does the plan **open with the core issue + core solution in plain English** (Move 0) — not dive straight into types?
- Does **every contract AND every step carry a plain-English twin** — the qualitative "what it's for" beside the technical "what it is"?
- Does **every boundary from `/brainstorm` have exactly one explicit contract**, defined once and referred to everywhere — not redefined per stage or passed as a bare dict?
- Does each contract name **who produces and who consumes** it, and keep **internal computation out** of the shared definition?
- Is **step one the contracts/data model**, with each later step depending only on earlier ones and **leaving the thing working**?
- Does **every step carry an inline check** (handoff to `/dataflow-verifier`) — no step without a way to prove it?
- Are the files/seams **real** (grounded in the codebase), and is the recordkeeping named rather than vaguely promised?
