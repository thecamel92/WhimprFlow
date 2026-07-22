# Model routing

## 1. Delegate the READING (the biggest lever)

Large reference files (specs, inventories, logs) are whole-system documents — any one
task needs a fraction of one. Reading them in the main loop puts the irrelevant
remainder into context **permanently**, re-sent and reasoned over on every subsequent
turn.

So: **send a subagent to read; keep only what the task needs.**

- Bulk reference reads, registry sweeps, "which files still do X" → a cheap subagent,
  returns the answer, not the file.
- A growing log/decision-record file should carry its own index (ID/date/title, no
  bodies) near the top — read that first, then a targeted `grep` + offset/limit `Read`
  for the one entry actually needed, rather than reading the file end-to-end.

**Extract verbatim what genuinely needs precision; summarizing is fine for bulk
lookups.** A subagent that paraphrases a spec/contract it was asked to relay is making
judgment calls it wasn't asked to make — nuance dropped in a summary is exactly the kind
of thing that later gets caught the hard way. Selection is safe; distillation of
something load-bearing is not.

## 2. Which tier

**Does the answer already exist somewhere, or does it have to be invented?**

- **Cheap/fast tier** — bulk reads, mechanical transcription, registry/bookkeeping
  updates, byte-identity checks. This is where the real cost gap is.
- **Mid tier** — conversions and ports with some judgment (preserving behavior/look
  while changing form); precisely-specified retunes. Transcription with judgment, not
  invention.
- **Top tier** — a genuinely new design/approach from a spec, work described only in
  prose ("make this feel more X"), first-time contract/shape extraction. The answer must
  be invented, and getting it wrong is often **silent** — it runs, it typechecks, and
  only a careful read catches it.

Practical check: *could someone competent but with zero judgment produce what's wanted
from this instruction alone?* If yes → cheap/mid tier. If the instruction would need the
words "use your judgment" → top tier.

## 3. Guards

1. **A cheap subagent does not self-certify a compliance/quality gate.** The gate is
   self-verification, and a gate that lies is worse than no gate. Re-run it in the main
   loop, or have the main loop check the diff.
2. **Rework costs more than the tier ever saves.** A correction means the whole task ran
   twice at the expensive tier — that dwarfs any saving from picking the cheap tier
   wrong the first time. When a design/architectural judgment gets corrected, write the
   correction into whatever spec/doc governs that decision, not just into memory — that
   turns the next similar task from invention into transcription.
3. **Context hygiene beats tier choice.** A stale, long conversation costs real money on
   every turn just to re-send — start fresh when a task is genuinely a new unit of work.
