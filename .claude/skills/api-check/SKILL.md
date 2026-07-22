---
name: api-check
description: STANDALONE READ audit — sweeps the codebase for third-party API / SDK / library usage and verifies each call against CURRENT vendor docs (via Context7) plus deprecation status, flagging anything stale, deprecated, or already removed and naming the replacement. The audit counterpart to the always-on Context7 rule: that rule stops NEW stale calls; this sweeps the OLD code for drift. General-purpose — audits ANY external dependency (payments, email, storage, LLM APIs, npm libs), not one vendor. Use when the user invokes /api-check, or asks "are my APIs current", "check for deprecated APIs", "am I using the correct/current API methods", "audit my third-party calls". Read-only — it reports, never edits.
---

# api-check

A **standalone, read-only audit** of the codebase's **external API surface** — every place your code depends on someone else's API, SDK, or library, checked against what that vendor currently ships.

It is the **audit counterpart to the always-on Context7 rule**: that rule stops you writing *new* stale calls; `api-check` sweeps the *existing* code for drift — the calls written before the rule, or that were fine once and have since been deprecated. And it's the **external sibling of `/dataflow-verifier`**: that traces/verifies the seams *inside* your system; this audits the seams where your system meets *someone else's*.

It **only reads.** It produces a report; it never edits and never migrates. Fixing a flagged API is a separate, deliberate step (usually plan mode, then the change).

The skill is **general** — it audits *every* third-party dependency the same way: cloud
LLM APIs (OpenAI/Anthropic-style), payment/email/storage SDKs, and any library/crate with
a breaking-change history. Don't tunnel-vision on the one obvious vendor; sweep the whole
external surface.

---

## The one failure mode this skill exists to prevent

**Silently depending on a deprecated or removed API** — code that works perfectly today and breaks the day the vendor pulls the endpoint or ages out the version. The failure is invisible until it's a production outage, because nothing in your own tests catches "the other side went away."

**The governing rule of this skill:**

> **Verify every external call against the vendor's CURRENT docs, not memory — and separate "works today" from "on a removal path."** A method that compiles and runs is not proof it's current; the vendor may have deprecated it with a removal date you haven't seen. Ground every verdict in live docs (Context7) or the vendor's changelog — never in what the model remembers.

---

## How to run it

### 1. Inventory the external surface
Grep/Explore for every third-party touch point — don't stop at the obvious one:
- **API calls** — REST endpoints, GraphQL mutation/query names, RPC method names.
- **The pinned version** — an `apiVersion` constant, a dated endpoint (`/api/2025-10/…`), an SDK version in `package.json`.
- **SDK / library imports** — client libraries whose methods can be deprecated across major versions.

Name the **real call sites** (file + line). Group by **provider**.

### 2. Pin the version in use, per provider
For each provider, find exactly what version the code targets — one centralised constant is the good case; scattered literals are a finding in themselves.

### 3. Verify each against current docs — with Context7
For each provider/call, use **Context7** (`resolve-library-id` → `query-docs`) to fetch the **current** surface and ask the direct question: *is this method/mutation/endpoint still current, or deprecated? Is this version still supported, or aging out?*
- **Prefer version-specific library IDs** when the vendor versions by date or major version (Context7 surfaces them).
- **For removal TIMELINES** (the actual dates), Context7's reference docs often won't carry them — **fall back to a web search of the vendor's changelog / deprecation notice.** State the date if you find one; say "no published removal date" if you don't.

### 4. Classify every finding
- **OK** — current, supported.
- **AGING** — still supported, but an old version heading toward end-of-life; bump recommended.
- **DEPRECATED** — a replacement exists and is recommended; migrate before removal.
- **REMOVED / BREAKING** — already switched off, or has a published removal date. Name the date.

For every non-OK finding, **name the replacement** (the current mutation/endpoint/method to move to).

### 5. Report — never edit
Output the audit and stop. Migrating a flagged API is a separate, deliberate change (hand off to plan mode), especially when it touches a high-consequence boundary.

---

## What the output looks like — the audit

A table, **most-severe first** (removed → deprecated → aging → ok):

| Provider | Call site | Version / method used | Status | Replacement |
|---|---|---|---|---|

Plus, per non-OK finding, a one-line recommendation and (if found) the removal date. And an **honesty line**: anything you could **not** verify (no Context7 coverage, no changelog found) is listed as *unverified*, not quietly assumed current.

---

## Pairs with

- **The always-on Context7 rule** — prevents *new* drift; `api-check` sweeps the *old*.
- **plan mode** — to plan the migration off any API this audit flags (the flagged API's current replacement is a clean external contract to plan around).
- **`/dataflow-verifier`** — the internal-seam sibling to this external-seam audit.

---

## Quick self-check

- Did you sweep the **whole external surface**, not just the obvious vendor?
- Is every verdict **grounded in live docs / a changelog** — not in memory?
- Did you separate **"works today"** from **"on a removal path"**, with dates where published?
- Does every non-OK finding **name its replacement**?
- Did you **list what you couldn't verify** rather than implying full coverage?
- Did you **only report** — no edits, no migration (that's a separate, deliberate step)?
