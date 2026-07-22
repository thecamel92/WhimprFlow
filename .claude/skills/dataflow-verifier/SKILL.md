---
name: dataflow-verifier
description: Traces and confirms the data crossing a boundary — the shape, its authoritative definition (or the finding that none exists), and whether it actually holds. Three modes — MAP (a whole unfamiliar area, comprehension only, nothing edited), AUDIT (one named seam, forensic depth: the authoritative shape, every rival/duplicate definition that disagrees with it, where the guarantee dies, whether illegal states are representable; no change required — this is the ground-truthing to do before a plan leans on a seam), VERIFY (a diff or just-edited files — verdicts every boundary the change touches SOUND / BROKEN / DEGRADES SILENTLY against what used to hold). An optional closing move authors the boundary assertion/stress test that locks a seam in. Use when the user invokes /dataflow-verifier, or asks "how does data flow through X", "what's the real shape of this contract", "is this seam duplicated anywhere", "does this change break the data flow", "add checks for this boundary". Has a matching `dataflow-verifier` AGENT — same three modes, read-only — for delegating a seam's ground-truth during planning (AUDIT) or spawning as the finalizing check at the tail of a pipeline (VERIFY). Pairs with /explain-simply for a plain-English rendering; grounds a seam for /brainstorm and /structure-plan before it becomes a contract, and implements the inline check each plan step carries.
---

# dataflow-verifier

One tool, three depths on the same question: **what crosses this boundary, and can it be trusted?** MAP answers it broadly, across a whole area you don't know yet. AUDIT answers it forensically, for one seam, whether or not anything has changed — the ground-truthing you do *before* a plan leans on a shape. VERIFY answers it differentially — did a specific change keep the promise a boundary used to make. The mechanism underneath is always the same (find the boundary, find what crosses it, confirm it holds); only **how many seams**, **how deep**, and **whether a change is the trigger** move.

> **Jargon, once:** a *boundary* (or *seam*) is any point where one part of the system hands data to another. What crosses it is its *shape*. A shape is **explicit** when a named type/contract defines it, or **implicit** when it's a bare dict/tuple/loose object passed on trust. The **authoritative definition** is the one place a shape is supposed to be defined; a **rival** is a second definition — a hand-written type sitting next to a validator, a duplicate interface under a different name — that quietly disagrees with it. A **silent-empty trap** is a boundary where "couldn't produce a value" and "produced nothing, legitimately" collapse into the same output, so the caller can't tell them apart.

---

## The one failure mode this skill exists to prevent

The same root cause bites at three different moments, wearing different clothes. Read a system without confirming its boundaries and you edit blind. Plan on top of a seam without confirming its truth and you build on a shape a rival definition already contradicts. Change something without re-checking and wrong-shaped data crosses silently, surfacing stages later as an unrelated-looking symptom. Every case is the same disease: **nobody actually confirmed what crosses the seam, so something confidently wrong took its place.**

**The governing rule of this skill:**

> **Find every boundary the task touches, get its shape from the code — never from memory, and never from the declared type alone — and confirm it holds.** Whether that means drawing a whole system, forensically pinning one seam's truth before you plan on it, or checking a specific change didn't quietly break what used to be true, the confirmation is not optional and it is never assumed from a type that merely compiles.

---

## Move 0 — Find the boundaries (shared foundation, every mode)

- **Delegate the wide reading.** Don't dump forty files into your own context — send a **`codebase-sweeper`** for any "which/where/how many," and a **`blast-radius`** when the real question is "if this changes, what breaks." They return answers, not files.
- **Identify the stages** data moves through (scrape → assemble → publish; request → handler → store — whatever this task's pipeline actually is).
- **For each boundary, record the pipe:** what crosses, producer → consumer, and whether the shape is **explicit** (a named type/contract) or **implicit** (a bare object passed on trust — flag it as a fragility regardless of mode).

---

## Pick the mode

- **MAP** — a whole area is unfamiliar, or the goal is genuinely just understanding. Comprehension only. **Nothing gets edited.**
- **AUDIT** — one named seam, no change required. This is the **ground-truthing** `/brainstorm` and `/structure-plan` reach for before a plan turns a seam into a contract: pin down the authoritative shape, and surface anyone who quietly disagrees with it.
- **VERIFY** — a diff or a set of just-edited files exists. Walk **only the boundaries the change touches** — both sides, including the far side that *didn't* change, since a break is often invisible from the side you're looking at.

A task can chain modes: AUDIT a seam to ground-truth it before planning a change, then VERIFY the change itself once it's built. Or MAP an unfamiliar area, then AUDIT the one seam inside it you're actually about to touch.

---

## Move 1 — Map (MAP mode)

- **Draw the spine:** `stage → [shape] → stage → [shape] → stage`, one line per stage on its role.
- **Flag every fragility:** implicit shapes, fields that could silently drop, silent-empty traps.
- Nothing here proposes a fix or touches code. A map is a document.

## Move 2 — Audit (AUDIT mode)

- **Find the authoritative definition** — if one exists. There may be none: the shape may exist only as whatever the producer happens to emit. **That's a finding, not a dead end** — say it plainly.
- **Hunt rivals by field names, not the type's name.** A duplicate is never spelled the same as the original — that's *why* it got written instead of reused. Check sibling files and shared/util modules first, since that's where the original usually is. Open both before calling anything a rival; similar names with different semantics aren't rivals.
- **Find where the guarantee dies:** `any`, an unchecked cast, `JSON.parse` straight into use, a `catch` that returns a partial object, an optional field every consumer dereferences unguarded.
- **Judge the shape itself:** are illegal states representable (a field combination that typechecks but can't actually exist)? Is there optionality that lies (marked `?` but every consumer treats it as mandatory)?
- **"One authoritative definition, honoured everywhere, no drift" is a valid and valuable verdict.** Say it plainly rather than manufacturing a rival that isn't real.

## Move 3 — Verify (VERIFY mode)

For each boundary the change touches:

- **Confirm the shape still matches what the far side expects.** Not "does this file typecheck" — does the *consumer*, possibly untouched, still get what it was built to receive?
- **Hunt specifically for:** a field silently dropped, a shape that used to be explicit and is now passed as a bare object, a "couldn't determine" state that now collapses into the same empty result as a legitimate empty.
- **Verdict every touched boundary:** `SOUND` / `BROKEN` / `DEGRADES SILENTLY`. The last category matters most — it's the one nothing else catches.
- **Every non-SOUND verdict needs a concrete failure scenario** — the actual input/state and what a user or the system would observe going wrong. "This might not hold" is not a finding.

## Move 4 — Lock it in (optional — on request, or when AUDIT/VERIFY finds a real gap)

Author the check that makes this boundary's failure loud and localized next time, matched to the risk:

- **Shape/contract mismatch** → a type-check or a boundary assertion.
- **Real logic** (a calculation, a branch) → a unit test with a couple of real cases.
- **UI/rendering** → run it and look — a passing type-check is not proof a screen renders.
- **Money-touching / must-never-happen** → an invariant guard that halts loudly.

Name the **exact command** to run and the **exact expected output.** This move closes a found gap — it isn't a mandatory test-writing quota.

---

## Chain-with-fallback

- **Chained:** `/brainstorm` and `/structure-plan` reach for **AUDIT mode** to ground-truth a seam before it becomes a contract; `/structure-plan`'s inline checks are exactly what **VERIFY mode** later confirms once the step is built.
- **Alone:** point it at an unfamiliar area (MAP), one seam (AUDIT), or an existing diff (VERIFY) — it infers the boundaries from the code either way, no upstream plan required.
- **In a pipeline/workflow:** spawn the **`dataflow-verifier` agent** in the matching mode instead of running this skill inline — AUDIT per seam during planning, VERIFY as the finalizing check after implementation. Same method, read-only, reports back without pulling the sweep into the main context.

---

## What the output looks like

- **MAP:** the spine diagram, per-seam detail (what crosses, producer → consumer, explicit/implicit), and the fragility flags.
- **AUDIT:** the authoritative definition (or "none — the shape is implicit"), the shape's fields marked guaranteed/assumed/lying-optional/unproven, every rival found, where the guarantee dies, and whether illegal states are representable.
- **VERIFY:** a verdict per touched boundary (SOUND / BROKEN / DEGRADES SILENTLY), with a concrete failure scenario for anything not SOUND.
- **Lock it in (if run):** the assertions/tests added, plus the exact run command and expected output.

---

## Quick self-check

- Did you **find boundaries by delegating the wide sweep**, not by reading dozens of files into your own context?
- In MAP mode, is the **spine actually drawable**, with every implicit shape and silent-empty trap flagged?
- In AUDIT mode, did you **search rivals by field name, not the type's name**, and open both sides before calling anything a rival?
- In VERIFY mode, did you check **both sides of every touched boundary** — not just the side that changed?
- Does every **non-SOUND / rival / illegal-state finding carry a concrete example** — not a vague "this might be an issue"?
- If you authored checks, did they **close a found gap** — not pad the diff with tests nobody needed?
