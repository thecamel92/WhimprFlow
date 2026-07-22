---
name: brainstorm
description: TECHNICAL altitude, step one — pressure-tests the SHAPE of a change before you commit to it. Names the boundaries and what data crosses them FIRST, then weighs 2–3 structural approaches (including framework choices like PydanticAI vs LangGraph) with tradeoffs stated, gives a plain recommendation, and flags the one-way doors. For a genuinely contested shape it can run several agents to propose and adversarially stress-test rival shapes. Broad strokes only — it does NOT define file layouts or exact data shapes (those belong to the plan itself). Use when the user invokes /brainstorm, or asks "what's the best shape/architecture for X", "which framework — A or B", "how should I structure this", "pressure-test this approach". Chain-with-fallback: feeds a chosen shape into planning.
---

# brainstorm

The **first technical move** — the floor directly below `/roadmap`. `/roadmap` has already chosen *what* to build next; brainstorm decides the **shape** it should take before a single line is committed. It works in **broad strokes**: the overall paradigm, the framework, the backbone — *not* the file layout or the exact data shapes, which belong to the plan itself.

The reflex this skill bolts on is the one a senior engineer makes without thinking and you don't yet: **name the seams before proposing a solution.** You cannot design a backbone until you know what flows through it — so dataflow is not a later concern here, it is *step one*.

> **Jargon, once:** a *"boundary"* (or *"seam"*, or *"pipe"*) is any point where one part of the system hands data to another — one stage to the next, one module calling another, the app talking to an external service. *"What crosses the boundary"* is the data that gets handed over at that point. A *"one-way door"* is a shape decision that's expensive to undo once other code depends on it.

---

## The one failure mode this skill exists to prevent

**Committing to a structure you never actually weighed** — jumping straight to the first plausible solution, then discovering three days into the build (or three days into the one big Claude Code prompt) that the shape was wrong and half the work has to unravel.

**The governing rule of this skill:**

> **Name the pipes first; propose shapes second.** Before any solution talk, list the stages and what data crosses each boundary. Only *then* put 2–3 structural shapes on the table, each with its cost stated, and flag which decisions are one-way doors. A brainstorm that opens with "let's use X" has already failed — it skipped the step where the shape gets weighed.

---

## How to run it

### 1. Name the boundaries and what crosses them — FIRST, always
Before proposing anything, draw the pipe: what are the **stages**, and what **data crosses between them**? For each seam, one line: *what goes in → what comes out.* This is your weakest reflex promoted to the opening move, because everything downstream (the backbone in the plan, the checks `/dataflow-verifier` runs) is built on these same boundaries. Get them named and the rest of the brainstorm has something real to hang on.

> **Name them from the CODE, not from memory.** Boundaries recalled rather than read are the imagined-codebase failure, and every shape chosen on top of a wrong seam is wrong with it. **Delegate the looking:** send a `dataflow-verifier` agent in AUDIT mode per seam you're unsure of (it returns the real shape and who re-declares it) and a `codebase-sweeper` for any "which/where/how many" question. They return the answer, not the files — so the sweep doesn't sit in context for the rest of the task. For a genuinely big change, don't run brainstorm piecemeal at all — the **`big-internal-plan` pipeline** (`.claude/pipelines/big-internal-plan.md`) conducts the whole process (recon sweep → debate → shape contest → contract/blast map → plan doc → critique) with these same agents as its engine.

### 2. Put 2–3 structural shapes on the table — with the tradeoff stated
For each candidate shape (this is where **framework choices live** — e.g. PydanticAI vs LangGraph vs hand-rolled; graph vs linear pipeline vs event-driven), state three things plainly:
- **What it buys** — the problem it solves well.
- **What it costs** — the complexity or constraint it imposes.
- **What it makes hard to reverse** — the one-way door it commits you to.

Keep it **broad strokes**. No file trees, no field-by-field data shapes — those belong to the plan. Here you're choosing the *kind* of backbone, not drawing it.

### 3. Give a plain recommendation
The user dislikes option-dumps. After laying out the shapes, **pick one and say why** — a clear leaning, not a survey. Name the runner-up and why it lost, so the choice reads as weighed, not defaulted.

### 4. Flag the one-way doors — and stop at them
Sort every decision into:
- **Reversible + cheap** → *"just try it in the build."* Don't agonise; the build will tell you.
- **Irreversible + expensive** (a data model, a shape everything will import, the framework the whole thing sits on) → **stop, decide deliberately, and reach for `AskUserQuestion`** rather than assuming. These are the user's calls to make; surfacing them is the single highest-value thing this skill does.

### 5. (Optional) Escalate a contested shape to several agents
When the shape is genuinely contested and high-stakes — the choice is expensive, the field is wide, and you're not confident — **run it as a contest** instead of deciding solo:
- Spawn **2–3 agents in parallel**, each championing a *different* rival shape, each returning: the shape, what it buys, what it costs, its worst failure mode.
- Then a **stress-test round**: attack each proposal on that worst failure mode (a fresh skeptic agent per shape, or a cross-examination pass) — *try to break it*, don't just admire it.
- **Synthesize:** present the survivors side by side, recommend one, and take the real fork to `AskUserQuestion`.

This is a scaling dial, not the default. A clear-cut shape needs step 3, not a fleet. Reach for the agents only when the fork actually deserves them.

---

## Chain-with-fallback

- **Chained:** the output — the named boundaries + the chosen shape — becomes the **opening context** of the plan (and later, of the one big Claude Code prompt). The boundaries you name here are the *same objects* the plan turns into contracts and `/dataflow-verifier` confirms hold. That single thread is what makes the moves reinforce instead of repeat.
- **Alone:** run it whenever a shape is unsettled, even without `/roadmap` upstream — it just asks you what the target is.

---

## What the output looks like — the brief

A short brief, chat-passed (nothing written to disk):
- **The boundaries** — the stages and what crosses each seam.
- **The chosen shape** — the paradigm/framework, in broad strokes, and why.
- **The rejected alternatives** — one line each, with why they lost.
- **The one-way doors** — flagged, with any that need a deliberate decision called out.

---

## Quick self-check

- Did you **name the boundaries before proposing any solution**? (If the brief opens with a framework, reorder.)
- Are there **2–3 real shapes**, each with *what it costs* and *what it makes hard to reverse* — not one shape dressed up as inevitable?
- Is there a **plain recommendation** with a named runner-up — not an option-dump?
- Are the **one-way doors flagged**, and were the expensive ones taken to `AskUserQuestion`?
- Did you stay in **broad strokes** — no file layouts, no exact data shapes (those belong to the plan)?
