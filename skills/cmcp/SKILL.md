---
name: cmcp
description: How to use cmcp — "curl for MCP" — a one-shot CLI that invokes any MCP tool (stdio or HTTP) from the terminal by reusing MCP server configs already installed for Claude Code, Claude Desktop, and Cursor. Use this skill whenever the user is iterating on / debugging an MCP server, wants to call a tool from a server that is not loaded into the current agent session, wants to inspect configured MCP servers or their tool schemas, or mentions keywords like "MCP", "mcp server", "mcp tool", "stdio server", "streamable-http", or asks to test an MCP tool from the shell — even if they don't explicitly say "cmcp".
---

# Using cmcp

`cmcp` is a command-line MCP client. Think `curl` for MCP: one invocation
opens a transport, speaks the MCP handshake, calls a tool, prints the
result, and exits. It auto-discovers server definitions from the user's
existing Agent configs (Claude Code, Claude Desktop, Cursor) plus its own
`~/.config/cmcp/mcp.json`, so you almost never have to configure anything
before calling a tool.

The killer use-case is the *dev loop*: rebuilding an MCP server every few
minutes without blowing away Claude Code's prompt cache, or exercising a
tool that isn't loaded into the current agent's tool list.

## When to reach for cmcp

Use cmcp when **any** of these are true:

- The user is developing their own MCP server and wants to call it after a rebuild.
- The user wants to call a tool on an MCP server that isn't loaded into the current agent session (e.g. you have no `mcp__foo__bar` tool but the server `foo` is configured in Claude Desktop).
- The user asks "what MCP servers / tools do I have configured?".
- The user wants to peek at a tool's input JSON schema before calling it.
- The user wants to pipe an MCP tool result into `jq` or another shell pipeline.
- You need stderr from a stdio MCP server to diagnose why it's misbehaving.

Don't use cmcp when the tool is already available as a native `mcp__*`
tool in this session — call it directly. cmcp is for when the native path
isn't available, isn't fresh, or the user explicitly wants a shell.

## Quick reference

```bash
# Discover
cmcp config sources                          # where configs are read from
cmcp server list                             # every server across all sources
cmcp server show <name>                      # resolved config (redacts secrets)
cmcp server show <name> --reveal             # unmask env / headers
cmcp tool list <server>                      # tools on a server
cmcp tool show <server> <tool>               # one tool's JSON schema

# Call a tool (three equivalent forms)
cmcp <server>/<tool>                         # positional shortcut
cmcp mcp://<server>/<tool>                   # explicit URI
cmcp call <server> <tool>                    # subcommand form

# Arguments
cmcp <server>/<tool> --arg key=value --arg other=123
cmcp <server>/<tool> --args-json '{"key":"value","nested":{"x":1}}'
# --arg values parse as JSON when possible (numbers, bools, arrays), else string.
# --arg overrides keys from --args-json when both are passed.

# Output & debugging
cmcp <server>/<tool> --json | jq '.content[0].text'   # raw CallToolResult JSON
cmcp -v <server>/<tool>                               # forward stdio server stderr
cmcp <server>/<tool> --timeout 60                     # per-call timeout (default 30s)

# Manage cmcp's own config (only ~/.config/cmcp/mcp.json is writable)
cmcp server add my-dev  --command node --arg /path/server.js --env DEBUG=1
cmcp server add my-api  --url https://mcp.example.com/mcp \
                        --header 'Authorization: Bearer xxx'
cmcp server remove my-dev
cmcp server edit                              # open $EDITOR on the file
```

## Decision tree

### "Run / call tool X on server Y"

1. If you're unsure the server exists: `cmcp server list`.
2. If you don't know the tool's arguments: `cmcp tool show <server> <tool>`
   and read its input schema.
3. Call it. Prefer the positional form `cmcp <server>/<tool>` — it's the
   shortest and matches the README's conventions. Pass structured input
   with `--args-json` when you already have a JSON object; use repeated
   `--arg key=value` for one or two scalar fields.
4. If the result needs to feed a shell pipeline or be stored as data, add
   `--json` so you get the raw `CallToolResult`. Without `--json`, cmcp
   pretty-prints for human reading.

### "My MCP server isn't responding / behaving weirdly"

1. `cmcp server show <name>` — confirm the command, args, env, or URL are
   what you expect. Pass `--reveal` only if you actually need to see
   secrets and the user is OK with that.
2. Call the tool with `-v` (`cmcp -v <server>/<tool> …`). For stdio
   servers this forwards the child process's stderr to your terminal,
   which is usually where the real error is.
3. If it hangs, raise `--timeout` or interrupt and inspect stderr.

### "What MCP stuff does this machine have configured?"

1. `cmcp config sources` — shows every config file cmcp merges, in
   priority order, and whether each exists. Priority (high → low):
   `~/.config/cmcp/mcp.json` → Claude Code (`~/.claude.json`) →
   Claude Desktop → Cursor. Higher priority wins on name collisions.
2. `cmcp server list` — the merged view. Each row shows which source
   the server came from.
3. Only cmcp's own file (`~/.config/cmcp/mcp.json`) is writable by
   `server add` / `remove` / `edit`. The others are read-only; to
   override an Agent-provided server locally, add an entry with the
   same name to cmcp's config and it wins.

### "Add a new server just for cmcp / override an Agent's entry"

Use `cmcp server add`. Two transport shapes:

- **stdio**: `--command <bin> --arg … --arg … --env KEY=VAL …` (repeat flags).
- **streamable-http**: `--url <https://…> --header 'Name: Value' …` (repeat `--header`).

These edits land in `~/.config/cmcp/mcp.json`. If you'd rather hand-edit,
`cmcp server edit` opens it in `$EDITOR` (creating a skeleton if absent).

### "I want to sandbox cmcp for a test"

Set `CMCP_CONFIG_DIR=/tmp/some-dir` — cmcp will read/write its own config
from there and ignore the default location. Agent configs are still
discovered from their canonical paths; if you want to hide those too,
point their env-vars / HOME at a scratch dir in the shell you spawn.

## Argument encoding

`--arg key=value` is the workhorse. Values are parsed as JSON when
possible, falling back to a string. This means:

- `--arg count=5` → integer `5`
- `--arg enabled=true` → boolean `true`
- `--arg tags='["a","b"]'` → array
- `--arg name=alice` → string `"alice"` (JSON parse fails, treated as string)
- `--arg name='"alice"'` → same string, via JSON path

Prefer `--args-json '{…}'` when the payload is nested or large — it's one
flag, one JSON blob, no shell-quoting surprises. Mixing is fine: start
from `--args-json` as the base and override individual keys with `--arg`.

## Output shape

Without `--json`, cmcp prints a human-readable rendering of the tool
result (text parts concatenated, non-text parts summarized). With
`--json`, you get the raw MCP `CallToolResult` object — this is what you
want when piping into `jq`, capturing to a file, or feeding another
program. The shape:

```json
{
  "content": [
    {"type": "text", "text": "…"},
    {"type": "image", "data": "…", "mimeType": "image/png"}
  ],
  "isError": false
}
```

`isError: true` means the tool itself reported an error — cmcp still
exits non-zero (code 4) so shell `&&` chains short-circuit.

## Exit codes

Use these in scripts / `&&` chains:

| Code | Meaning                              |
|-----:|--------------------------------------|
| 0    | success                              |
| 1    | generic IO / JSON error              |
| 2    | config or invalid-URI error          |
| 3    | server not found / transport failure |
| 4    | tool not found / tool-reported error |
| 5    | timeout                              |

## Examples

**Call a configured tool, parse its text output:**
```bash
cmcp cortex/list_projects --json | jq -r '.content[0].text'
```

**Debug a stdio server you're developing:**
```bash
cmcp -v my-dev/do_thing --arg path=/tmp/x --timeout 60
```

**Override a Claude-Code-provided server locally without touching Claude:**
```bash
cmcp server add github --command node --arg /path/to/my/fork/index.js
cmcp server show github          # now resolves to your fork
```

**Remote streamable-http server with bearer auth:**
```bash
cmcp server add api --url https://mcp.example.com/mcp \
                    --header 'Authorization: Bearer sk-...'
cmcp tool list api
cmcp api/search --arg q=rust --json | jq '.content'
```

## Gotchas

- **SSE and OAuth are not supported** (MVP). Streamable-HTTP with a
  static Bearer header works; interactive OAuth does not yet.
- **Secrets are redacted by default** in `server show`. Don't assume a
  blank-looking env var is actually empty — re-run with `--reveal` if
  you need to verify.
- **Name collisions** resolve by priority, not by "last one wins" at
  runtime. If `cmcp server list` shows an unexpected command for a
  server, check `cmcp config sources` to see which file is winning.
- **Prefer native `mcp__*` tools** when they exist in the current
  session — they're faster (persistent connection) and don't spawn a
  child process per call. Reach for cmcp when the native path is
  missing, stale (post-rebuild), or the user explicitly wants shell.
