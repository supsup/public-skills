---
name: claude-api-full
description: Opt-in access to the FULL bundled Claude API reference library (~60k-token injection, or file-granular reads) for library-grade work — writing/migrating substantial API code, SDK integrations, Managed Agents builds — where cross-referencing breadth beats per-page fetches. The lite `claude-api` cheatsheet shadows the bundled skill by default; this skill is the sanctioned way back to the deep reference when a session genuinely needs it.
---

# claude-api-full — the library, on purpose

The bundled Claude Code `claude-api` reference is name-shadowed by the lite cheatsheet
(deliberate — the ~60k injection was firing on two-fact questions; see the README in this
skill pair's repository). This skill is for the flows that genuinely WANT the library:
building or migrating real API code, SDK integrations, Managed Agents work — sessions
already costing 100k+ where one authoritative 60k beats eight serial page-fetches plus
retry loops.

Two ways in — pick by how much you need:

## A. Full-fidelity restore (the whole 60k, exactly as shipped)

The skill index hot-reloads on filesystem changes, so this works mid-session:

```bash
mv ~/.claude/skills/claude-api ~/.claude/skills/claude-api.off   # un-shadow
# invoke the `claude-api` skill NOW — it resolves to the bundled reference
mv ~/.claude/skills/claude-api.off ~/.claude/skills/claude-api   # re-shadow when done
```

Do the restore, invoke, and re-shadow in the SAME session; never leave the shadow off
(that silently re-opens the 60k-per-fact-lookup tax for every future context).

## B. File-granular reads (3–30k, pick your topics)

The bundled reference's files live unpacked in the Claude Code bundled-skills cache
(location varies by platform; typically under the system temp directory). Locate:

```bash
find "${TMPDIR:-/tmp}" /private/tmp -type d -name claude-api -path '*bundled-skills*' 2>/dev/null
```

Then `Read` only what the task needs from that directory:

| Task | Files |
|---|---|
| SDK code in language X | `<lang>/claude-api/README.md` (+ `tool-use.md`, `streaming.md`, `batches.md`, `files-api.md` as needed) |
| Model migration | `shared/model-migration.md` (large ~35k — the whole procedure; consider WebFetching the live page instead for freshness) |
| Tool use / agents design | `shared/tool-use-concepts.md`, `shared/agent-design.md` |
| Prompt caching deep-dive | `shared/prompt-caching.md` |
| Managed Agents | `shared/managed-agents-overview.md` first, then the topical `shared/managed-agents-*.md` |
| Errors / models / platforms | `shared/error-codes.md`, `shared/models.md`, `shared/platform-availability.md` |
| Live doc URLs | `shared/live-sources.md` |

Caveats: the cache path is version-suffixed and temp-dir-resident — it can vanish on
reboot or CLI update. If absent, use route A, or WebFetch the live pages from the lite
skill's escalation table. File-granular reads miss the composed guidance prose (defaults,
API-drift table) that only exists in the injected form — for anything drift-sensitive,
prefer route A or the live migration guide.
