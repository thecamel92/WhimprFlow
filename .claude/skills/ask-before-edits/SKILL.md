---
name: ask-before-edits
description: Turn on "ask before edits" mode — a session-long mode where every code edit is explained in very high detail AND proposed as the edit in the SAME turn, so the explanation and the diff's split panel appear together and the user's Accept/Reject of that panel IS the approval (there is NO separate verbal go-ahead step). Use when the user invokes /ask-before-edits, says "turn on ask before edits", "explain edits before making them", or asks to review/approve each change with a deep explanation. The user is newer to TypeScript and wants to learn from each change.
---

# Ask before edits

A session-long **mode**. The user is learning the codebase and wants to understand — and approve — every change as it's proposed. While this mode is ON, you do **not** silently apply edits, and you do **not** split the explanation from the code into two rounds. For each edit, in **ONE turn** you write the in-depth explanation **and** make the edit call — so the explanation and the diff's split panel appear **together**. The user approves or rejects **in the panel**; that Accept/Reject **is** the go-ahead. **Never stop after an explanation-only turn to wait for a verbal "yes"** — that is the exact behaviour this mode must avoid.

This amplifies the standing preference to over-explain to a TypeScript beginner (see CLAUDE.md). When in doubt, explain more, not less.

## Detailed BUT simple — the golden rule

"Very high detail" does **not** mean "dense and technical." The user has **little coding experience**, so an explanation that's thorough but full of jargon has failed. Every explanation must be both **complete** and **easy to understand**:

- Write in **plain English**, short sentences. Imagine explaining to a smart friend who has never coded.
- **Don't assume jargon is known.** If you must use a coding term (`type`, `prop`, `async`, `const`, `interface`, `state`, `CSS variable`…), define it in a few plain words the first time, or use an everyday **analogy** ("a type is like a label that says what kind of value is allowed here").
- Prefer "what it does and why it matters" over "what it's technically called."
- Depth comes from **covering all three angles below clearly**, not from cramming in advanced terminology. If a sentence wouldn't make sense to a beginner, rewrite it simpler.

## Turning the mode on / off

- **ON:** the moment this skill is invoked (e.g. `/ask-before-edits`), the mode is active for the **rest of the session** — across every subsequent task — until the user turns it off. Triggering it puts the session into **'manual' mode** — every edit must be **manually approved** through the split-panel diff (Accept/Reject); edits are NOT auto-accepted. If the session is currently on auto-accept, say so and ask the user to switch to manual, or the split panel won't appear.
- **OFF:** when the user says something like "turn off ask before edits", "stop asking", "you can just edit now", or "exit ask-before-edits mode". Acknowledge that the mode is off and resume normal editing.
- If you're unsure whether it's still on, assume it **is** and keep explaining — the cost of an extra explanation is low; a surprise unexplained edit defeats the purpose.

## The protocol for EVERY edit

The explanation and the real edit must appear **together, in the same turn** — so the user reads the plain-English explanation and sees the actual before→after diff **at the same moment**, not one after the other. Do NOT explain in one turn and wait for a verbal "yes" before making the edit in a later turn — that separation (the old behaviour) forced the user to picture the code from words alone. Instead:

1. **Write the three-part explanation first** (as text — the format is below), naming the file as a clickable link. This is the whole point — be genuinely detailed, pitched at a TypeScript beginner:
   - **1. The technical coding aspect** — what the code *actually does*, in plain words. Explain the pieces and *why it's written this way*, defining any coding term as you go. (e.g. instead of just "this is an optional property `?:`", say: "the little `?` means this value is optional — the code works whether or not it's filled in"; instead of "`??` vs `||`", say: "this picks a backup value only when nothing was set"). Keep it accurate but un-intimidating.
   - **2. What it means by itself** — the change's standalone purpose and effect, in everyday language. What does this edit *accomplish* on its own? What will look or behave differently because of it?
   - **3. How it ties into the broader code** — the bigger picture, told as a simple story. What other files / parts of the app use or rely on this? What does it connect to before and after it? Name them with links. Mention anything that must stay true (an "invariant" = a promise the code relies on) and any knock-on effects — explained plainly.
2. **In the SAME message, immediately make the `Edit` / `Write` / `NotebookEdit` call.** Don't wait for a separate verbal go-ahead first. The editor's own diff panel — the before→after view with its **Accept / Reject** buttons — is what shows the code and collects approval, so it opens with your explanation sitting right above it. The user reads the words and reviews the real diff **side by side**.
3. **The panel's Accept / Reject IS the go-ahead.** If the user rejects the edit (or asks for a change), revise and re-issue the explanation + edit **together** again. Approving one edit is **not** blanket approval for the rest — keep pairing each subsequent edit with its own explanation (you may flow through an already-approved batch without re-asking, see below).

> **Note — this relies on edits prompting for approval.** The pairing only works if the diff panel actually appears (i.e. edits are set to ask, not auto-accept). If the user has auto-accept on, the edit would apply instantly with no panel — so keep edits in an approval-prompting mode while this skill is active. If an edit ever applies without a panel, tell the user so they can switch the mode back.

## Presentation format

Put the explanation immediately before the tool call, in this shape, then make the edit in the **same** turn:

> **Edit N — `path/to/file.ts`** — one-line summary of the change
>
> **1. Technical:** …
> **2. On its own:** …
> **3. Ties into:** …

You don't need to paste the full before→after code in the text — the diff panel that opens with the edit already shows it. A short inline snippet is fine only when it helps the explanation point at one specific line.

## Batching (keep it practical, not maddening)

- A single logical change that spans **several edits to the same file** can be presented as one batch: write each edit's three-part explanation, each immediately followed by its own `Edit` call, so every explanation still sits right above its own diff panel. Don't collapse them into one panel with one lumped explanation.
- Independent changes across **different files** should each get their own explanation-then-edit pairing. If a task needs many files, work through them in groups — don't fire 15 edits at once behind a wall of text; keep each panel next to the words that explain it.
- For a **trivial, zero-judgement** edit (a typo fix, a rename the user explicitly dictated), a short single-paragraph explanation is fine — but still pair it with the edit so the panel and explanation appear together. Match the depth to the edit's real complexity.

## What counts as an "edit"

Any file mutation: `Edit`, `Write`, `NotebookEdit`, or a `Bash` command that writes/moves/deletes project files. Read-only work (searching, reading, running typechecks/tests, `git status`) does **not** need this treatment — do it freely so your explanations are well-grounded.

## Still flag high-consequence edits

This mode is **in addition to**, not instead of, calling out edits that are unusually
consequential or hard to reverse — shared data contracts/boundaries, native/FFI or
OS-permission code, auth/secrets handling, anything touching persisted user data. For
those, give a brief What / Importance / Why / Challenge heads-up *and* the three-part
explanation.
