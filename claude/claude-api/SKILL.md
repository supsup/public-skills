---
name: claude-api
description: Verified-fact cheatsheet for Claude API lookups — model IDs/pricing/context/output caps, image-token accounting, prompt-caching economics, current-model 400 traps. DELIBERATELY SHADOWS the bundled claude-api reference to avoid its ~60k-token injection on two-fact questions. NOT sufficient for writing/migrating API code, SDK bindings, or Managed Agents work — for those, follow the ESCALATION table inside to WebFetch the specific docs page. Never answer from an expired block or for a model not listed; escalate instead.
---

# Claude API — fact cheatsheet (lite shadow of the bundled reference)

**What this is:** the high-frequency facts, ~2k tokens instead of the bundled ~60k
injection. **What this is not:** a coding reference. If the task is *writing or migrating
API code, SDK bindings, structured outputs, tools, or Managed Agents*, this file is
INSUFFICIENT by design — jump to **Escalation** below. To restore the full bundled skill
permanently: delete or rename `~/.claude/skills/claude-api/`.

**Consequence-bound escalation (mechanical, not judgment):** live-verify against the
source URL — regardless of stamp age — whenever (a) a price is about to be QUOTED to a
person or used in a billing decision, (b) a model ID or parameter is about to be WRITTEN
into code/config, (c) the model isn't in the table below (NEVER guess an ID), or
(d) the block is past its `expires`. Reading one docs page ≈ 3–10k tokens; still 6–20×
cheaper than the bundled injection.

## 1. Models & pricing  `verified: 2026-07-14 · expires: 2026-07-28 · source: platform.claude.com/docs/en/about-claude/models/overview.md + /docs/en/pricing.md`

| Model | ID (exact, no date suffixes) | Context | Max output | In $/MTok | Out $/MTok |
|---|---|---|---|---|---|
| Claude Fable 5 | `claude-fable-5` | 1M | 128K | 10.00 | 50.00 |
| Claude Mythos 5 (Project Glasswing only) | `claude-mythos-5` | 1M | 128K | 10.00 | 50.00 |
| Claude Opus 4.8 | `claude-opus-4-8` | 1M | 128K | 5.00 | 25.00 |
| Claude Opus 4.7 | `claude-opus-4-7` | 1M | 128K | 5.00 | 25.00 |
| Claude Opus 4.6 | `claude-opus-4-6` | 1M | 128K | 5.00 | 25.00 |
| Claude Sonnet 5 | `claude-sonnet-5` | 1M | 128K | 3.00 (intro 2.00 → 2026-08-31) | 15.00 (intro 10.00) |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | 1M | 128K | 3.00 | 15.00 |
| Claude Haiku 4.5 | `claude-haiku-4-5` | 200K | 64K | 1.00 | 5.00 |

- Cache read ≈ **0.1×** input price; cache write **1.25×** (5-min TTL) / **2×** (1-hour TTL).
- **Batch API = 50% off** all token usage (async, ≤24h).
- **`count_tokens` endpoint is FREE** — never estimate, never tiktoken.
- Rate limits (RPM/TPM/tiers): **deliberately not cached here** — account-specific and
  volatile; check the Console or `/docs/en/api/rate-limits.md`.

## 2. Image / vision token accounting  `verified: 2026-07-14 · expires: 2026-09-12 · source: platform.claude.com/docs/en/build-with-claude/vision.md`

- Images are billed as 28×28-px patches: **tokens = ⌈w/28⌉ × ⌈h/28⌉** (~1 token per 784 px).
- Same per-token price as text input. No image discount.
- High-res tier (Fable 5 / Mythos 5 / Opus 4.8 / 4.7 / Sonnet 5): max **2576 px long edge,
  4784-token cap** per image. Standard tier (all others): 1568 px / 1568 tokens. Bigger
  images are DOWNSCALED to fit (small text becomes illegible — that's the failure mode).
- Hard limits: 8000×8000 px per image; 10MB/image on the API; >20 images per request
  tightens the per-image limit to 2000 px.

## 3. Prompt caching  `verified: 2026-07-14 · expires: 2026-09-12 · source: platform.claude.com/docs/en/build-with-claude/prompt-caching.md`

- **Byte-exact prefix match**; render order `tools → system → messages`; any byte change
  invalidates everything after it. Max **4** `cache_control` breakpoints.
- `{"type":"ephemeral"}` = 5-min TTL (1.25× write); `{"type":"ephemeral","ttl":"1h"}` =
  1-hour TTL (2× write). Reads ≈ 0.1×.
- Minimum cacheable prefix (below it, silently no cache): Opus 4.8/4.7/4.6/4.5 + Haiku 4.5
  = **4096** tokens; Fable 5 / Sonnet 4.6 = **2048**; Sonnet 4.5 = **1024**.
- Breakpoints look back at most **20 content blocks** for a prior cache entry.
- Verify with `usage.cache_read_input_tokens` — zero across repeats = silent invalidator
  (timestamps, unsorted JSON, varying tool sets).

## 4. Current-model 400 traps  `verified: 2026-07-14 · expires: 2026-07-28 · source: platform.claude.com/docs/en/about-claude/models/migration-guide.md`

One line each — these COMPILE in your head but 400 on the wire:
- `temperature` / `top_p` / `top_k`: removed on Fable 5, Opus 4.8/4.7 (Sonnet 5 rejects
  non-default values). Omit them.
- `thinking: {type:"enabled", budget_tokens:N}`: removed on Fable 5 / Opus 4.8/4.7 /
  Sonnet 5 → use `{type:"adaptive"}`. Explicit `{type:"disabled"}` 400s on Fable 5 — omit
  the param instead.
- Last-assistant-turn **prefill**: 400 on the whole 4.6+ family and Fable 5 → use
  `output_config.format` (structured outputs).
- Fable 5 requires **30-day data retention** — a ZDR org gets 400 on every request.
- Effort lives at `output_config: {effort: low|medium|high|xhigh|max}` (not top-level).

## Escalation table (this file is NOT enough for these)

| Task shape | WebFetch |
|---|---|
| Write/modify API code, SDK usage | `platform.claude.com/docs/en/api/messages/create` + the SDK repo (`github.com/anthropics/anthropic-sdk-<lang>`) |
| Model migration (breaking changes) | `platform.claude.com/docs/en/about-claude/models/migration-guide.md` |
| Tool use / structured outputs | `platform.claude.com/docs/en/agents-and-tools/tool-use/overview.md`, `/docs/en/build-with-claude/structured-outputs.md` |
| Managed Agents (sessions/vaults/deployments) | `platform.claude.com/docs/en/managed-agents/overview.md` (+ topic pages under `/managed-agents/`) |
| Extended/adaptive thinking, effort | `/docs/en/build-with-claude/adaptive-thinking.md`, `/effort.md` |
| Streaming, batches, files, PDFs, citations | `/docs/en/build-with-claude/streaming.md`, `/batch-processing.md`, `/files.md`, `/pdf-support.md`, `/citations.md` |
| Errors / rate limits | `/docs/en/api/errors.md`, `/docs/en/api/rate-limits.md` |
| Latest models/pricing | `/docs/en/about-claude/models/overview.md`, `/docs/en/pricing.md` |

**Library-grade work** (building/migrating substantial API code, SDK integrations,
Managed Agents builds — where cross-referencing breadth genuinely beats per-page
fetches): invoke the **`claude-api-full`** skill. It documents the sanctioned full-60k
restore (un-shadow → invoke → re-shadow, hot-reload makes it mid-session-safe) and the
file-granular alternative (Read 1–5 topical files from the bundled cache, 3–30k).

## Freshness & failure protocol

- **Refresh ritual** (owner: whoever installed this): re-verify + re-stamp every block on
  any Anthropic model launch, or by each block's `expires` date, whichever first.
- **Shadow-integrity check, same ritual + on any Claude Code update:** shadowing binds by
  NAME. Verify the bundled reference is still named `claude-api` (check the skill index
  or the bundled-skills cache under your system temp dir) — if a CLI update renamed it,
  this shadow silently stops preempting and the ~60k injection returns; rename this
  directory to match the new name.
- **First lie = quarantine, not patch:** if any block contradicts a live source, mark
  that block `QUARANTINED — do not serve` in place, answer from the live source, and
  record the incident wherever you track such things. Retire the whole skill (delete the
  dir, restoring the bundled reference) only if the failure is systemic — refresh ritual
  or trigger contract broke, not one stale number.
