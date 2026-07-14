# Taming Claude Code's bundled `claude-api` skill (a ~60k-token injection) with a two-tier shadow

## The problem

Claude Code ships a built-in `claude-api` skill — a comprehensive reference for building
against the Claude API. It's genuinely good documentation, but it has a cost profile
that's easy to miss:

1. **Its trigger is broad and mandatory-toned.** The skill's description (always present
   in the model's context) tells Claude to load it before answering *any* LLM-shaped
   question — model names, pricing, token limits, caching, or any agent/RAG/tool-use task
   with no provider named — and never to answer those from memory.
2. **Loading it injects the entire reference library at once** — the full model-migration
   guide, a dozen-plus Managed Agents documents, per-language SDK docs. In our
   measurements: **~55–60k tokens per invocation**.
3. **Injected content compounds.** Skill payloads become part of the conversation
   context, so they are re-read on every subsequent turn of an agent run. A 60k injection
   in a 20-turn agent session costs far more than 60k.

The result: every fresh context (each new session, each subagent) that so much as brushes
an LLM-shaped topic pays the full library price — usually to answer a two-fact question
like "what's the image token formula?"

**Measured impact:** we ran the same multi-turn agent task twice — once with the bundled
skill, once with the lite shadow below. Same inputs, same instructions:

| | Bundled skill | Lite shadow |
|---|---|---|
| Total tokens | 246k | **58k** |
| Tool calls | 7 | 20 (did *more* work) |

~4.2× cheaper, because removing the 60k injection also removed its per-turn re-read.

## Where it bites you: four everyday flows

**1. The two-fact question.**

> **You:** "Roughly what would a million input tokens of Haiku cost?"
> **Claude:** *(trigger fires: pricing question → loads the bundled skill → 60k tokens of
> migration guides, Managed Agents docs, and seven languages of SDK bindings arrive)*
> "$1.00 — Haiku 4.5 is $1/MTok input."
>
> You paid for the encyclopedia; the answer was one table cell. And the 60k now rides in
> your session context for every following turn.

**2. The innocent mention.**

> **You:** "Summarize this PR for me." *(the PR diff happens to touch a config file
> containing `model: claude-opus-4-8`)*
> **Claude:** *(the trigger fires on the mere mention of a model name → 60k injection)*
> "This PR refactors the retry logic…"
>
> The task needed zero API knowledge. The trigger can't tell "uses the API" from
> "contains the string" — mentions of Claude/Anthropic/model names anywhere in the
> working set are enough.

**3. The subagent fan-out.**

> **You:** "Spawn three agents to review these services in parallel."
> **Each agent, independently:** *(fresh context, brushes an LLM-shaped detail → its own
> 60k load)*
>
> Skill loads don't share across contexts. Three subagents = three full injections —
> ~180k tokens and real money in cache writes before any of them has done a minute of
> actual review work.

**4. The long-run compounding (the expensive one).**

> **Your agent, turn 2 of 20:** *(loads the bundled skill for a passing question)*
> **Turns 3 through 20:** *(the 60k payload is part of the conversation now — it is
> re-sent and re-read with every single request)*
>
> This is how a "60k skill" becomes the dominant line item of a 250k agent run. The
> injection isn't a one-time toll; it's a passenger that never gets out of the car. In
> our A/B above, removing it cut the identical task from 246k to 58k — far more than 60k
> saved, because the compounding went with it.

## The solution: name-shadowing + a two-tier design

Claude Code's documented skill precedence makes this fixable without touching the
bundled skill: **a user skill with the same name overrides a bundled skill**
([docs](https://code.claude.com/docs/en/skills.md), "Where skills live" — "A skill at any
of these levels also overrides a bundled skill with the same name"). There is no
per-skill disable for bundled skills (only the all-or-nothing `disableBundledSkills`), so
shadowing is the mechanism.

This directory contains two skills:

### `claude-api/` — the lite shadow (default path, ~1.8k tokens)

A verified-fact cheatsheet that shadows the bundled skill by name. It covers the four
high-frequency lookup categories:

1. Current model IDs, context windows, output caps, pricing (incl. cache multipliers)
2. Image/vision token accounting (the patch formula, per-tier resolution caps)
3. Prompt-caching economics (read/write multipliers, TTLs, minimum cacheable prefixes)
4. Common 400 traps on current models (removed parameters, prefill, retention rules)

The design principles, learned the hard way:

- **A stale cached fact is worse than an expensive correct one.** Every fact block
  carries `verified:` and absolute `expires:` dates plus a `source:` URL. Volatile blocks
  (models/pricing) get short horizons; stable formulas get longer ones.
- **Escalation is bound to consequences, not judgment.** If a price is about to be quoted
  to a person, a model ID is about to be written into code, the model isn't in the table,
  or a block is expired — the skill instructs Claude to live-verify against the source
  URL regardless of how fresh the stamp looks. Fetching one docs page (~3–10k tokens) is
  still far cheaper than the library.
- **First lie → quarantine, not silent patching.** If a block contradicts a live source,
  it gets marked QUARANTINED in place (with the incident recorded wherever you track such
  things), and the whole skill is retired only if the failure is systemic.
- **Unknown model IDs are never guessed.** Not in the table → escalate.

### `claude-api-full/` — the librarian (opt-in path back to the library)

Shadowing makes the bundled reference unreachable by name — but some flows *genuinely
want* the full 60k: writing substantial API integration code, running a model migration,
building on Managed Agents. There, one authoritative library beats eight serial page
fetches. This skill documents the two sanctioned routes back:

- **Full restore:** temporarily rename the shadow directory away, invoke `claude-api`
  (which now resolves to the bundled skill), and rename it back when done. The skill
  index hot-reloads on filesystem changes, so this works mid-session.
- **File-granular reads:** the bundled reference's individual files sit unpacked in
  Claude Code's bundled-skills cache; reading 1–5 topical files costs 3–30k instead
  of 60k. (The cache directory contains only the reference files — the injected prompt
  body is composed inside the CLI at invoke time, which is also why you can't fork the
  full skill by copying the directory.)

## Install

```bash
cp -R claude-api claude-api-full ~/.claude/skills/
```

That's it — skill resolution picks up the shadow immediately (no restart; we observed
the index hot-reload even mid-session, including for subagent spawns).

To uninstall / restore the stock behavior: `rm -r ~/.claude/skills/claude-api
~/.claude/skills/claude-api-full`.

## …and where the shadow bites you instead

Fair's fair — the trade-offs, as flows:

**1. The deep-work session that starts shallow.**

> **You:** "Build me a TypeScript service that streams Claude responses with tool use."
> **Claude:** *(loads the lite shadow → 1.8k → reads its own boundary rule: "NOT
> sufficient for writing API code")* — and now it must escalate: fetch the tool-use page,
> the streaming page, maybe the SDK repo.
>
> Three or four page fetches later you've spent ~25k and gotten paraphrased-from-live-docs
> answers instead of the curated library. For genuine build sessions, invoke
> `claude-api-full` up front — that's exactly what it's for. The failure mode is
> forgetting it exists and letting the agent drip-fetch.

**2. The expired block that refuses to answer.**

> **You:** *(six weeks after installing, having never re-stamped)* "What does Sonnet cost?"
> **Claude:** "The pricing block in my reference expired on \<date\> — fetching the live
> pricing page to be sure…" *(+8k tokens)*
>
> Annoying — and correct. A skill that answered anyway from a stale table would be worse.
> But every escalation-on-expiry is a tax you chose by not re-stamping; if you see this
> often, do the five-minute refresh.

**3. The silent un-shadowing.**

> **A Claude Code update ships. The bundled skill is now named `anthropic-api`.**
> **Your next session:** *(the shadow no longer matches any bundled name; the renamed
> bundled skill surfaces with its broad trigger → the 60k injection is quietly back)*
>
> Nothing errors. Nothing warns. Your token bills just drift up. This is the one failure
> mode with no loud signal, which is why the shadow-integrity check is written into the
> lite skill's refresh ritual — after CLI updates, glance at the skill index.

**4. The stale-fact near-miss (the one that matters).**

> **You:** "Add the model to prod config."
> **Claude:** *(model table says X; the table is three weeks past a model launch you
> didn't re-stamp for)*
>
> This is the catastrophic version — a wrong fact served confidently. It's why the
> consequence-bound escalation exists (model IDs about to enter code must be live-verified
> *regardless of stamp age*) and why the protocol on a caught lie is quarantine, not a
> quiet fix. If you strip those rules to save tokens, you've rebuilt the original problem
> with worse data.

## Maintenance — read this part

You are taking over freshness responsibility from Anthropic's bundled skill, which
updates with every CLI release. Three obligations:

1. **Re-verify and re-stamp the fact blocks** on every Anthropic model launch, or by each
   block's `expires` date, whichever comes first. The source URLs are in each block.
2. **Check the shadow still binds after Claude Code updates.** Shadowing binds by NAME —
   if a future CLI release renames the bundled skill, your shadow silently stops
   preempting and the 60k injection returns. The check is written into the lite skill's
   refresh ritual.
3. **Honor the quarantine protocol.** The fact tables in this repo were verified on the
   date stamped in each block; treat them as a starting snapshot, not a source of truth.

## Known limits

- The shadow only exists where the skills directory exists: containerized or remote
  Claude instances with a different home directory still pay the bundled price.
- An agent can still reconstruct the 60k voluntarily by reading the entire bundled cache;
  the skill instructs one-file-at-a-time, but that's instruction, not enforcement.
- The measurements above are from mid-2026 Claude Code versions; both the bundled skill's
  size and the precedence behavior may change.
