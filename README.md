# public-skills

Skills we run in production on our own agent harnesses, published as-is. Each entry links
to a directory containing the skill file(s) and a README covering the problem it solves,
the design, installation, and — where it applies — the measured impact and the honest
trade-offs. Licensed under [Apache 2.0](LICENSE).

## Claude (Claude Code)

### [`claude-api` + `claude-api-full`](claude/) — tame the bundled API reference's ~60k-token injection

Claude Code's built-in `claude-api` skill fires on almost any LLM-shaped question — a
model name in a diff is enough — and injects its entire ~60k-token reference library
into the context. Because injected content is re-read on every subsequent turn, that one
load compounds into the dominant line item of long agent runs, and every fresh subagent
pays it again. In our A/B measurement, an identical multi-turn agent task cost **246k
tokens with the bundled skill vs 58k with this replacement** (~4.2× cheaper, doing more
work).

This pair fixes it with Claude Code's documented skill-precedence rule (a same-named
user skill shadows a bundled one):

- **`claude-api`** — a ~1.8k-token verified-fact cheatsheet that shadows the bundled
  skill: current model IDs/pricing/context caps, image-token accounting, prompt-caching
  economics, and current-model 400 traps. Every fact block carries `verified`/`expires`
  dates and a source URL, with mechanical escalation rules (prices about to be quoted or
  model IDs about to enter code get live-verified regardless of stamp age; unknown models
  are never guessed) and a quarantine-on-first-lie protocol.
- **`claude-api-full`** — the sanctioned road back to the full library for the sessions
  that genuinely want it (writing substantial API code, migrations, Managed Agents
  builds): a mid-session un-shadow/re-shadow recipe plus a file-granular reading map of
  the bundled reference.

**Use it if** your sessions or subagents keep paying a five-figure token toll to answer
two-fact questions about the Claude API. **Read the maintenance section first** — you're
taking over fact-freshness from Anthropic's update cycle, and the README documents
exactly where that bites if you don't.

→ Full write-up, walkthrough flows, install, and caveats: [`claude/README.md`](claude/README.md)

## OpenAI (Codex)

Nothing published yet.
