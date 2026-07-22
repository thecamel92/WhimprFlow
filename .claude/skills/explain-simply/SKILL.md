---
name: explain-simply
description: OVERLAY (couple with any skill, or run alone) — a plain-English mode that explains a plan, a dataflow, or a chunk of code to someone newer to coding. The read-side twin of ask-before-edits (which explains EDITS as they happen); this explains what's being built or what already exists, on demand. Defines jargon on first use in plain, literal terms (the BRIDGE — not real-world/situational analogies), KEEPS crucial technical specifics named rather than hiding them, names the real pieces paired with what each one DOES, and tells the story of the data end to end. Use when the user invokes /explain-simply, says "explain this simply", "like I've never coded", "break this down", "what does this actually do", or couples it with /roadmap, /brainstorm, /dataflow-verifier, or plan mode.
---

# explain-simply

An **overlay**, not a phase. It clips onto any of the other skills — or runs on its own against any plan, dataflow, or chunk of code — and renders it in plain English for someone with real vision but hackathon-level coding experience, not AWS-engineer-level.

It is the **read-side twin of `ask-before-edits`**: that skill explains *edits* as they happen; this one explains *what's being built or what already exists*, so you can actually evaluate a plan or understand a system instead of nodding along to it.

It **only explains.** It never edits, never plans, never decides — it makes something already on the table understandable.

---

## The one failure mode this skill exists to prevent

**Nodding along to something you can't actually evaluate** — approving a plan, or believing you understand a system, when the jargon slid past you and the dataflow never became a picture. Silent incomprehension is expensive: it's how a wrong shape gets approved and only bites three days later.

**The governing rule of this skill:**

> **Every technical term is defined the first time it appears, in plain literal language — the bridge, not a real-world analogy — and every named piece is paired with what it *does*. Crucial specifics are NAMED and explained, never sanded off to make the sentence smoother.** An explanation that uses a word the reader doesn't have, names a file without saying what it's for, OR quietly drops a load-bearing detail (a contract, a boundary, a key field) to seem simpler, has failed no matter how smooth it reads.

---

## The golden rules

- **Plain English, short sentences.** Explain it the way you would to a smart friend who has never coded. Depth comes from *clarity and coverage*, not from fancier terminology.
- **Define jargon the first time, in plain literal terms — the bridge, not a real-world analogy.** Just unpack what the word means *here*, directly. e.g. *"a `type` says what kind of value is allowed in a given spot"; "a boundary is a point where one part of the code hands data to another"; "an invariant is a rule the code assumes stays true."* The reader can follow plain language fine — they only need the term itself unpacked. **Skip everyday-scenario analogies** (mailboxes, kitchens, address books); don't map the mechanism onto an unrelated situation.
- **Keep the crucial specifics — name them, don't sand them off.** Simplifying means making the real thing understandable, NOT omitting it. If a detail is load-bearing — a named contract, a specific boundary, a key function or field — **name it and explain it plainly**: give the real name, then its plain-terms job in parentheses (e.g. "the *X* contract (the agreed shape stage A hands to stage B)"). Never swap a crucial specific for a vague hand-wave to smooth the sentence — explain it in simple terms, but still call it by its real name. Simpler wording, same real detail.
- **Name the real pieces — always paired with what they do.** Reference the actual files (as clickable links) and the key functions/values, but every name comes with its plain-terms job, so the reader can connect the explanation to the real thing without already knowing it.
- **Tell the story of the data.** What gets touched, in what order, how the parts connect, and where the risky seams are. A dataflow told as a story ("the URL comes in here, gets turned into *this*, which the next part reads to make *that*") lands where a list of components doesn't.
- **Ground it in the real thing first.** Read the actual code/plan before explaining it — no hand-waving, no invented pieces.

---

## Formats it can reach for

Pick whatever makes *this* thing clearest — not all of them, every time:
- **A plain-English narrative** — the story of the change or the system.
- **An ASCII dataflow diagram** — `input → [what it becomes] → next part → [what it becomes] → output`, when the shape is the hard part.
- **A mini-glossary** — a few terms defined in plain literal terms, when the thing is jargon-dense.

---

## Coupling — how it overlays the others

- **On a plan or `/brainstorm`:** apply the golden rules to that output — translate the contracts, the shapes, the one-way doors into plain terms *as they're produced*, so you can evaluate the plan while it's being written.
- **On `/roadmap`:** render the sequencing reasoning (dependencies, one-way doors) in plain English.
- **On `/dataflow-verifier`:** turn the traced pipes (MAP mode) or the boundary verdicts (VERIFY mode) into the story of how the system actually moves data.
- **Alone:** point it at any code, plan, or concept and ask "what does this actually do?" — it's a pure explainer with no side effects.

---

## Quick self-check

- Did **every piece of jargon** get defined on first use in plain literal terms (no situational analogies)?
- Were the **crucial specifics kept and named** — every load-bearing contract/boundary/field called by its real name, explained simply, not sanded off?
- Is **every named file/function paired with what it does** in plain terms?
- Did the **dataflow come across as a story** — input to output — not just a parts list?
- Was it **grounded in the real thing** you read, with no invented pieces?
- Did it **only explain** — no editing, planning, or deciding slipped in?
