# big-internal-plan — planning pipeline for ambitious INTERNAL changes

> **Scope:** ambitious changes to **this codebase only** — internal operations. A change that involves **external applications/services** as the actual subject of the work (not just a detail) is outside this pipeline's scope — treat it as a separate, smaller conversation rather than stretching this one over it.

## What this is (and isn't)

This is a **playbook, not a script**. Each stage below gives the general direction and the right tools for it — how many agents, which kinds, what they're for. Adapt freely: merge stages, reorder them, loop back, run a stage twice, spin out a dozen agents at once when the problem warrants it. Problems at this scale are big and irregular; a rigid procedure would fit them worse than judgment does.

Only three anchors don't bend:

1. **Evidence over memory.** Every claim about the codebase in the final plan traces to an agent finding or a direct read. Planning against an imagined codebase is the one failure this pipeline exists to prevent.
2. **The user stays in the loop.** Questions, clarifications, debate, and the real decisions (the shape, the one-way doors) happen with them, in the main window — as many rounds as it takes, at whatever stage they arise. Never bury a genuine fork inside an agent. Run this as a running **`/grill-me`**: one question at a time, your recommendation on the table for each, threaded through every stage as it surfaces something only they can decide — not saved up into one big round at the end.
3. **The plan is a document, and it gets attacked before it's done.** Fresh-context critics read the file cold; the author never self-certifies.

Everything else — the stage order, the agent counts, the exact document sections — is guidance.

## The toolbox

- **The four agents** (via the `Agent` tool with `subagent_type`, or inside a workflow via `agentType`) — they return **answers, not files**, keeping the main window clean. Their model tiers are deliberate — match the tier to whether a wrong answer is loud or silent:
  - `codebase-sweeper` — "which / where / how many / what deviates" inventory questions. The bulk of every fan-out; cheap by design, so many at once is fine.
  - `dataflow-verifier` in **AUDIT mode** — "what shape actually crosses this seam, and who re-declares it". Tracing is reading; a missed detail usually surfaces loudly later (type-checker, plan gates). (The same agent's VERIFY mode confirms an implementation later — not used during this planning-only pipeline.)
  - `blast-radius` (top tier) — "if this changes, what breaks" across build-time, runtime, cross-process/external-contract, and already-persisted data. Top tier because its whole value is the consumers grep can't see, and a missed consumer is a *silent* failure. Spend it on real trace questions, not bulk lookups.
  - `plan-critic` (**unpinned — inherits the session model**) — fresh-context attack on a shape or a drafted plan; give it the artifact, never the conversation. Unpinned on purpose: the attacker must never be weaker than the author it's attacking — a weaker critic's missed findings are unrecoverable.
- **Ad-hoc workflows** — for a genuinely huge batch (a fan-out big enough to want one deterministic call that runs everything, chains follow-ups in the background, and reports every gap), you may author a one-off `Workflow` script inline for that moment. Optional — direct `Agent` spawns are the default and are equally valid.
- **The gap ledger** — however the agents are spawned, keep track of every question/trace sent out. Anything that returned nothing, or was never asked, goes into the plan's **Unchecked** section — a gap reported loudly is fine; a gap swallowed silently is not.
- **The skill bricks** — `/brainstorm`'s moves for weighing shapes, the contract + foundation-first moves for the plan body, `/plan-stress-test`'s grounding bar for judging critiques.

## The stages — general direction, not a checklist

### 1 · Gather
Goal: know the real codebase before shaping anything. **The primary data-gathering is a fan-out of `codebase-sweeper` agents** — several at once is normal, many is fine (they're a cheap tier, so the cost lives there, not in the main window). Mix in `blast-radius` where the question is "who breaks" and `dataflow-verifier` (AUDIT mode) where it's "what crosses this seam". Other agent types are fine too, but keep the *bulk* reading on these cheaper agents — don't pay main-loop rates to grep. Derive the questions from the user's problem statement plus whatever design/decision docs the project keeps — and while deriving them, also write down the questions **only the user can answer** (intent, priorities, tolerances, which behaviours must survive); those seed stage 2.

### 2 · Questions & debate — with the user
Ask the questions — **all of them, and no shame in a lot of detailed ones**: an unasked question becomes a guessed premise that surfaces three stages later as a wrong plan. It does not need to be all at once, between stages 1 and 3. You should aim to ask questions throughout the planning process. `AskUserQuestion` in batches for crisp forks; `/grill-me` for anything more open — one question at a time, your recommendation offered first, waiting for their answer before the next. Push back where their answers conflict with the evidence. This isn't a single gate — the `/grill-me` thread stays open at *any* stage, whenever something genuinely hangs on the answer, so clarification happens as it's needed rather than in one dump. Ordering with stage 3 is situational: when the problem is already precise, shaping first and debating over concrete attacked options often works better than abstract questions.

### 3 · Shape
Put real rival shapes on the table (built from the evidence, honest about costs), and **attack them before recommending** — `plan-critic` per shape, `blast-radius` on each shape's concrete change surfaces. Then `/brainstorm`'s closing moves: recommend one plainly, name the runner-up and why it lost, and take the expensive one-way doors to the user. When rival shapes aren't obvious, three priors usually span the space: **minimal-diff** (the smallest change that genuinely solves it), **contract-first** (fix the data model/seams properly even if wider), **redesign** (replace the mechanism the problem lives in, migration included). Inventing shapes is judgment work — do it in the main window or at the session model; keep the *attacks* on the cheaper agents.

### 4 · Deep map
For the chosen shape: `dataflow-verifier` (AUDIT mode) per seam it crosses, `blast-radius` per thing it changes — each trace ending in a file-by-file list. This is what makes the final document's contracts and blast table **read, not recalled**.

### 5 · Author the plan — main window, user watching
Write the document (convention: `plans/<YYYY-MM-DD>-<slug>.md`, create `plans/` if missing) from the accumulated evidence. **Scale it to the problem** — a genuinely big change earns a long, detailed document; never compress it to fit a template, and never pad a small one to look thorough. If you catch yourself writing a file path or contract no agent reported, that's a gap: verify it or mark it unchecked.

### 6 · Critique
Send fresh-context `plan-critic`s at the drafted file — they `Read` it cold, through distinct lenses (contracts / missed consumers / sequencing & migration are good defaults; add problem-specific ones). Judge each finding: confirmed → revise; rejected → record why in the document (a rejected finding with its reason is future-proofing, not noise). Loop as needed; when a disagreement won't resolve, it goes to the user, not to more agents.

## The plan document

Cover what the problem demands — these are the sections that usually earn their place in a plan this size; scale, merge, or skip deliberately (and say so) rather than by omission:

- **Framing** — the problem and the idea of the fix, in plain English a non-specialist can judge.
- **Decision record** — the questions asked, the user's answers, the premises they fixed; plus critic findings that were rejected, with why.
- **Chosen shape & rejected rivals** — the winner in broad strokes; each rival one line with the reason it lost; one-way doors marked *user-confirmed*.
- **Contracts** — one authoritative shape per boundary, producer/consumer named, wire-vs-internal split, each with a plain-English twin.
- **Blast radius, by file** — file → what changes → why → risk, including the consumers grep can't see (an on-disk format an older version wrote, a value another process reads).
- **Ordered steps** — foundation-first, each independently shippable, each naming real files/seams, each with its inline check.
- **Risks, one-way doors & accepted losses** — and what would force a replan.
- **Verification plan** — the `/dataflow-verifier` handoff.
- **Unchecked** — every angle the pipeline failed to cover (dead agents, capped fan-outs, unverified claims).
- **Evidence appendix** — the distilled findings the plan leans on, so the doc stands alone.
- **Recordkeeping** — which docs (if the project keeps any) get entries on acceptance, drafted here, written only once accepted.

Two of these are **never skipped silently**: the file-by-file blast radius (the whole reason the map stage exists) and **Unchecked** (an absent Unchecked section is a claim of completeness — make the claim honestly or list the gaps).

## Working notes

- **Cost sanity:** bulk reading and mechanical verification on the cheaper agents; the expensive tiers go where wrong answers are silent (blast traces, critiques) or where judgment lives — the debate, the shape call, the authoring, all in the main window (see `.claude/rules/models.md`).
- **Resumability:** every stage is independent — if a fan-out dies or the session breaks, redo just that piece; the plan doc on disk carries the state forward.
- For moderate tasks, skip the pipeline entirely — `/brainstorm` → plan mode inline is the right size.
