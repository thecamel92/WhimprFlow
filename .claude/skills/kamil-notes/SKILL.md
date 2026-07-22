---
name: kamil-notes
description: Write a concise note into Kamil's personal notes file (kamil-notes.md). Use when the user invokes /kamil-notes, or says "note this to kamil-notes", "add to my notes", "record this in kamil-notes". Write-only — it appends/updates notes; it does not read them back unless separately asked.
---

# kamil-notes

Append a **concise** note to the user's private scratchpad at `kamil-notes.md`. This file is Kamil's long-term memory of how the project works — connections between pieces, and what specific functions/files do. It is gitignored and only touched when he asks.

## What to write

- The note is whatever the user just gave you (a fact, a connection, an explanation of a function/file). Capture **the point**, not everything.
- Focus on **connections** and **specific functions / files** — not documentation.

## Note structure (required format)

Every note follows this exact shape — the notes file is an **ordered, numbered list**:

1. **Number** — the next number in sequence (continue from the last note in the file).
2. **Title** — a short note title.
3. **Relevant files** — the important file(s) only. If the thing spans many files, list the key ones and write **"and more"** — do NOT enumerate every file.
4. **What it does** — what the function/piece does, and (if relevant) **how it connects to other functions**.
5. **Snippet** — the relevant snippet(s) of code only (a line or few — never a full block).
6. **Data flow** — what **output** the function produces (a snippet of the output shape + a brief qualitative explanation of what it is), and **which function + file ingests it** next. This traces the pipe: producer → shape → consumer.

Template to follow:

```
## N. <Title>
**Files:** `path/to/a.ts`, `path/to/b.ts` (and more)
<what it does — plus how it connects to other functions, if relevant>
`<relevant one-line snippet, if it helps>`
**Flow:** outputs `<output shape/snippet>` (<what it is, briefly>) → consumed by `<fn>` in `path/to/consumer.ts`
```

## Rules (non-negotiable)

1. **Keep it concise. Never bloat the file.** A few lines. If it's getting long, cut it.
2. **Only the relevant snippet.** When noting code, the user does NOT care about every argument — capture the one-line snippet that makes the point (or describe it in words). **No big code blocks.**
3. **Correct mistakes before writing.** If the user's dictation has a wrong technical detail, say plainly what's off and why, give the correct version, and save the **accurate** note — never transcribe the error. (He's newer to TypeScript and relies on this.)
4. **Update, don't duplicate.** If a related note already exists in the file, extend/fix that one instead of adding a second.
5. **Match the file's style.** Short `##` heading, tight prose, `inline code` for names, links like `[file.ts](path)` where useful.

## How to do it

1. Read `kamil-notes.md` (needed anyway to edit it, and to check the style + for an existing related note).
2. Draft the note applying the rules above. If you corrected something, mention the correction to the user.
3. `Edit` the file — append under a new `##` heading, or update the existing related note.
4. Confirm briefly what you saved (one line). Don't re-dump the whole note.

## Don't

- Don't touch this file unless invoked (via `/kamil-notes` or an explicit "note this to kamil-notes").
- Don't add it to git, the project's docs, or persistent memory — it's local and personal.
- Don't pad with headers, preamble, or restated context. Brevity is the whole point.
