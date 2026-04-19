---
name: synap
description: How to use Synap — a local-first structured memory engine exposed via MCP — for long-term user memory, ad-hoc app namespaces (tasks, notes, accounting, knowledge bases), and hybrid keyword + semantic search. Use this skill whenever the user asks you to remember something about them, recall prior context, create a new kind of data store on the fly, search across past notes, or wherever the tools `mcp__synap__remember`, `mcp__synap__query`, `mcp__synap__search`, `mcp__synap__init_app`, `mcp__synap__write`, `mcp__synap__list_apps`, `mcp__synap__read_file`, `mcp__synap__forget`, or `mcp__synap__expand_context` would apply — even if the user doesn't explicitly mention "synap".
---

# Using Synap

Synap is a local-first memory engine for AI agents. Everything lives under
`~/.synap/` on the user's machine — no cloud, no accounts, no API keys.
You talk to it via the `mcp__synap__*` tool family.

This skill teaches you the decision tree: **which tool, when, with what
arguments**. Read the tool schemas for exact parameter syntax; this doc is
about choosing the right tool and using it well.

## Core concepts

Synap organizes data into **apps**. An app is a namespace (`tasks`,
`accounting`, `recipes`, …) with a schema. Schemas are EAV under the hood, so
you can add fields later without a migration. Each app holds **entities**
(rows) with **attributes** (fields).

Two apps are built-in and owned by Synap itself:

- `synap.memories` — long-term memory about the user. Writes go through
  `remember` / `forget` (never `write`). Reads go through `query` / `search`.
- `synap.sessions` — conversation history (turns, summaries). Read-only for
  you; don't write to it.

User-facing apps you or the user create live under other names (no
`synap.` prefix — that namespace is reserved).

## Decision tree

### "Remember X about me"

Use `remember` against `synap.memories`. Two shapes:

- **Profile attribute** (`type: "attr"`) — a persistent fact about the user
  (`preferred_language`, `city`, `allergies`). Use `op: "set"` to overwrite,
  `op: "append"` to add to a list.
- **Event** (`type: "event"`) — a timestamped observation ("user shipped
  v0.1.0 today", "user mentioned feeling overwhelmed"). Append-only, with
  `description`, `subject`, optional `when`.

Pick **attr** when the fact is enduring state. Pick **event** when it's a
moment in time that matters for context later.

### "What do you know about me?"

`query` on `synap.memories` with `filters=[{field:"subject", op:"eq",
value:"user"}]`. Don't use `search` unless the user asked something fuzzy
like "anything about my travel plans" — then `search` with `rank:
"time_decay"` surfaces recent relevant events.

### "Forget X"

`forget`. Precise, keyed. Don't "query then write nulls" — `forget` handles
both attr clearing and event deletion atomically.

### "Track tasks / recipes / expenses / …" (new kind of data)

1. `list_apps` — check if one already exists.
2. If not: `init_app` with a thoughtful schema.
3. `write` to add entities. `ops` is an array: **one op = single write,
   N ops = atomic batch** (all-or-nothing).

Schema design tips:
- **Single-type shorthand**: pass `fields: [...]` (instead of `types:
  {<name>: [...]}`) when the app holds one shape of entity. The type
  name becomes implicit — queries can omit `type` entirely.
- Mark long-text fields (descriptions, notes) `vectorized: true` so search
  can find them semantically.
- Use `required: true` and `default: <value>` on fields you want the
  server to validate or fill (e.g., `required` on `amount`, `default:
  "JPY"` on `currency`).
- Use `types` (plural) if one app holds different shapes ("task" +
  "note" in one `workspace` app). Inside each type, set `type: "..."`
  on create ops.
- Don't over-design v1 — EAV lets you add fields later.

### "Show me my todos / last week's notes / …"

`query` with filters and sort. TSV output is dense — parse it, don't dump
it back to the user raw.

### "Find anything about X"

`search`. Three ranking modes:
- `relevance` (default) — pure semantic + keyword match.
- `recency` — newest first, ignoring similarity. Good for "latest notes".
- `time_decay` — blend of both. Good for evolving context (events,
  journals) where both freshness and relevance matter.

Pass `app` to scope, or omit for cross-app search.

### "Read this file"

`read_file` — handles `file://`, `synap://` URIs, and local paths. Returns
chunked pages (default ~4K chars). Use `page` parameter to page through.

### "What was that you told me earlier?" (context has been compacted)

Look for `<compressed_segment turn_ids="...">` blocks in the conversation.
`expand_context` with those `turn_ids` drills back to the verbatim
originals. `<stale_tool_result turn_id="...">` is the same pattern for
folded tool output.

## Anti-patterns

- **Don't write to `synap.*` apps.** Synap rejects it. Use `remember` /
  `forget` for memory.
- **Don't reinvent memory with a custom app.** `synap.memories` has
  profile+event semantics, time-decay ranking, and pre-turn injection
  built in.
- **Don't over-chunk manually.** Long text (>1024 tokens) is auto-chunked
  with overlap. Just write the whole thing.
- **Don't ignore the subject field.** Memory is scoped by `subject`
  (defaults to `"user"`). If the user is tracking someone else ("remember
  Alice prefers tea"), pass `subject: "alice"`.

## Common patterns

### Setting up a new area of the user's life

User: "I want to start tracking my reading."

```
1. list_apps — confirm no "reading" app exists
2. init_app app="reading" with fields:
   - title (string, required: true)
   - author (string)
   - status (enum: to_read | reading | done, default: "to_read")
   - started_at, finished_at (string — ISO 8601)
   - notes (string, vectorized: true)
   - rating (number)
3. "Ready — tell me what you're reading and I'll add it."
```

### Surfacing context at the start of a conversation

The `[memory: profile]` and `[memory: events]` blocks are injected for you
automatically every turn. You don't need to call `query` to see them. But
if the user asks something that needs broader recall ("what have I been
working on lately?"), `search` on `synap.memories` with `rank:
"time_decay"` is the right call.

### Migrating / reshaping data

Synap has no formal migration. To add a field: just start writing it.
Existing rows simply won't have the attribute. To rename: `query` to get
all entity_ids, then `write` a batch of updates mapping old field → new
field → clear old. EAV cost is minimal.

## Tool cheat sheet

| Goal | Tool | Notes |
|------|------|-------|
| Remember a user fact | `remember` type=attr | `op: set` or `append` |
| Log a timestamped event | `remember` type=event | append-only |
| Clear a memory | `forget` | precise key, don't guess |
| Create a data app | `init_app` | `fields:[...]` single-type, `types:{...}` multi-type |
| List apps | `list_apps` | `include_internal: true` to see `synap.*` |
| Add/update entities | `write` | ops array, N = atomic batch |
| Filter entities | `query` | TSV output; project with `select:[...]` |
| Semantic find | `search` | rank: relevance \| recency \| time_decay |
| Read a file | `read_file` | pages automatically |
| Recover compacted turns | `expand_context` | pass turn_ids from the compressed block |
| Detect links between entities | `detect_links` | opt-in, for knowledge graphs |
| Embedding queue depth | `embed_status` | check progress after `reindex` |
| Re-embed vectorized fields | `reindex` | scope with `app` / `type` / `field` |
