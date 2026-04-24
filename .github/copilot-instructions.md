# MemPalace — Copilot Instructions

This repository is MemPalace itself. A MemPalace MCP server may be active in this session. Use it.

> For broader project context, philosophy, and design principles, see [CLAUDE.md](../CLAUDE.md) and [AGENTS.md](../AGENTS.md).

## When to search memory

Call `mempalace_search` **before** answering any question that touches:

- prior decisions (architecture, naming, API shape, rejected alternatives)
- people (contributors, collaborators, stakeholders)
- project history (what was built, what changed, why something was removed)
- open or resolved work items, plans, or roadmap items
- anything the user has previously told you about this codebase or their preferences

Do not guess. If memory is potentially relevant, search first, then answer. A missed memory is a broken promise.

## How to search

```
mempalace_search("<natural-language query>")
```

Use plain-language queries. The tool performs hybrid BM25 + vector search and returns verbatim stored text. Do not paraphrase results — surface them as-is to preserve accuracy.

To orient yourself in the palace before a deep search:

```
mempalace_status()           # palace size, wing/room breakdown
mempalace_get_taxonomy()     # full wing → room → drawer count tree
```

## When to save information

Call `mempalace_add_drawer` to file verbatim content when:

- a decision is made and the reasoning matters
- the user shares context, preferences, or constraints that should persist
- a non-obvious fact about the codebase is established

Always check `mempalace_check_duplicate` before filing to avoid duplicates.

File content **verbatim**. Never summarize, paraphrase, or compress what the user said.

## MemPalace constraints — non-negotiable

**Verbatim always.** Content stored in MemPalace is the user's exact words. When you retrieve a drawer, return its text as-is. Do not rewrite it.

**Local-first.** All memory operations happen locally on the user's machine through the MCP server. No cloud calls, no external services, no API keys are required or permitted for memory operations.

**No lossy compression.** The index layer uses AAAK compression for fast scanning, but the drawers themselves are never summarized. The AAAK index points to full verbatim drawers — it does not replace them.

## Palace structure

```
WING   — person or project (e.g. "alice", "mempalace-backend")
  ROOM — day or topic grouping (e.g. "2026-04-23", "search-design")
    DRAWER — verbatim text chunk (the actual stored content)
```

Use `mempalace_list_wings`, `mempalace_list_rooms`, and `mempalace_list_drawers` to navigate the structure when you need to inspect a specific wing or room.

## Knowledge graph

For structured entity relationships, use:

- `mempalace_kg_query` — query entity relationships
- `mempalace_kg_add` — add a relationship
- `mempalace_kg_timeline` — temporal view of relationships

