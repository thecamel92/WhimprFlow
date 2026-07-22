# big-internal-execution — the perimeter around building an ambitious INTERNAL change

> **Scope:** *building* a change to **this codebase only** — the downstream counterpart to **`big-internal-plan`** (which decides *what* to build and hands over a plan doc). Work involving **external applications/services** (a third-party API/infra as the subject of the work rather than a detail) has **no execution pipeline** — don't stretch this one over it.

## What this is — and emphatically is NOT

**This pipeline does not tell you how to build.** That is the entire design. Execution runs exactly as it would without it: the same skills, the same judgment, the same order, the same compliance/quality gates the project has (if any), the same CLAUDE.md rules. **If you are reading this file to decide how to write the code, the pipeline has failed** — close it and go work.

What it adds is a **perimeter**: a small fixed set of checks that fire *around* otherwise-untouched work, and a **gate** at the end the user actually reads before anything is recorded as done. `big-internal-plan` is heavy because *shaping* is where judgment compounds. Building is not that — building is where **wrapping** helps and **supervising** hurts.

Three anchors, and only three:

1. **Execution is untouched.** No stages between you and the code, no mandated order, no agent looking over your shoulder. Build the way you would anyway.
2. **The perimeter fires ONCE, at the end** — not per-edit, not per-file. It is a check on finished work, not a process to build inside of. A perimeter that interrupts is a perimeter that gets disabled.
3. **Nothing is recorded until the user confirms.** `docs-updater` fires on **their word**, never on Claude's assessment that things went well.

## The toolbox

- **During the build — standby only:** `codebase-sweeper`. Reach for it *only* if you genuinely need data you don't have ("which files still do X", "where is Y read"). It is not a checkpoint; it's there so you never work a step against an imagined codebase. Zero calls is a perfectly normal run.
- **At the perimeter — both, in parallel, once:**
  - `dataflow-verifier` in **VERIFY mode** — every boundary the change touched, verdicted SOUND / BROKEN / DEGRADES SILENTLY. Catches the break that still typechecks.
  - `silent-failure-hunter` — swallowed errors and silent failure paths in the diff. Catches the failure that still *runs*.
  - Run them **concurrently** — they're independent, both read-only, and neither needs the other's answer.
- **After confirmation only:** `docs-updater` — mechanical transcription into the project's docs/changelog, from the entries the gate drafted and the user approved.

---

## 1 · Build

Normally. Untouched. `codebase-sweeper` on standby if a real data gap appears.

## 2 · Perimeter

When you believe you're done: `dataflow-verifier` (VERIFY) + `silent-failure-hunter`, in parallel, on the diff. Fix what they find — then re-run only what you invalidated.

## 3 · The gate — main window, never delegated

**Why this stage exists.** `docs-updater` is about to write an entry that means *"delivered and **verified**."* Confirm on a feeling and that entry is false — and unlike a bug, it never surfaces. It sits in the record that every future plan reads as ground truth, and `big-internal-plan`'s first anchor ("evidence over memory") quietly starts drawing from a poisoned well. Per `.claude/rules/models.md` guard #1, **a gate that lies is worse than no gate** — so this runs in the main window and is never handed to a subagent that would self-certify it.

**The rule:** **separate what you PROVED from what you CLAIMED from what nobody CHECKED — and never let the third column be silent.** An absent "unchecked" list is itself a claim: it says *everything was covered*. Make that claim honestly, or list the gaps.

**a · Run what already exists (the regression question).** Whatever the project's real commands are — its full test suite and its full typecheck/lint, not a filtered or targeted subset:
- **The whole test suite** — not a targeted subset; the regression you didn't predict is in the file you didn't think to name.
- **The full, unfiltered typecheck/build.** Do **not** lean on a session-scoped hook that only checks files edited this session — a break in a consumer you never opened is invisible to that, and that consumer is precisely the break this gate exists to catch.
- **Read the results precisely** — this is where a gate lies to itself. **Red = a blocker** (top of the ledger, not a footnote). **Green ≠ the feature works** — it proves *the cases someone already thought to write* still pass, and earns **nothing** in PROVEN. **No test touches this change at all** → an UNCHECKED entry, stated plainly.
- **Don't author tests to fill the gap here.** A coverage gap is a *finding* — put it in the ledger and let the user call it. If a boundary needs locking in, that's `dataflow-verifier`'s Move 4 and it belongs there. Writing tests on their say-so is fine; writing them because the gate felt thin is how a suite fills with tests nobody trusts.

**b · Drive it (the proof question).** A typecheck proves a shape is internally consistent; tests prove the cases someone thought to write. **Neither proves the thing works.** Reach for the project's own run/verify path (or the built-in `/run`) to exercise the change end-to-end — don't hand-roll a driving procedure that already exists. What you *observe* here is the only thing that earns **PROVEN**. Not "typecheck passed." *"I ran the app, triggered the changed path, and observed X"* — that's proof. If the change genuinely has no runtime surface (a docs edit), say so: *"there was nothing to run"* and *"I didn't run it"* are different sentences, and the first must never stand in for the second.

**c · Collect and re-check.** Read the perimeter's verdicts (a `DEGRADES SILENTLY` is the headline, not a footnote). Then: **any compliance/quality gate a subagent self-certified does not count** — re-check it here against the actual diff, or say plainly in the ledger that it remains self-certified and unverified.

**d · Name the standing obligations** — surface them, don't perform them. Whatever this project's own equivalents are: a stale generated artifact that now needs regenerating (a one-line nudge, never fired automatically unless the user has said to), doc entries owed but not yet written (draft what should be written and where; write nothing — that's stage 4), and any other "these two things must move together" rule the project has.

**e · Present the ledger, then STOP.** Hand the user the picture and **ask**. Don't assess the task as done — that's not your call, and pre-judging it is how a rubber stamp gets manufactured. **Lead with the weakest column**, never a victory lap. **No padding**: if everything was proven and nothing is outstanding, say it in three lines and ask — manufactured concern teaches them to skim the gate, which recreates the exact failure this stage prevents.

```
PROVEN (I ran it and observed this)
  <what was driven, and what actually happened>

CLAIMED (the code says so; nobody watched it)
  <what should work, and why it wasn't observed>

UNCHECKED (nobody looked)
  <every gap — checks not run, agents that returned nothing,
   self-certified gates not re-verified, angles never covered>

STANDING OBLIGATIONS
  <stale artifacts to regenerate (nudge ONLY) · doc entries drafted, unwritten · other coupling rules>

→ Confirm and I'll record it, or tell me what to close first.
```

## 4 · Record

On the user's confirmation — and **only** then — `docs-updater` writes the drafted entries.

---

## Working notes

- **A finding is not a failure.** `DEGRADES SILENTLY` is precisely what the perimeter is for; it means the perimeter worked. Fix it and move on — don't treat it as a verdict on the build.
- **Don't run this on a trivial change.** A one-line copy tweak doesn't need two agents and a ledger. This pipeline is for changes big enough that "I think it's done" is a claim worth testing.
- **Cost sanity:** the perimeter is a couple of mid-tier agents and a cheap scribe. The expensive tier is spent where it belongs — the building itself, and the gate that reads the result.
