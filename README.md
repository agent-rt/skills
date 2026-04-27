# agent-rt/skills

Agent Skills curated by [agent-rt](https://github.com/agent-rt).

A Skill is a folder with a `SKILL.md` file — YAML frontmatter + Markdown
instructions — that an AI agent loads on demand to handle a specific task
well. See [agentskills.io](https://agentskills.io) for the spec.

## Install via the agentskills.io ecosystem

Install every skill in this repo into your local agent:

```sh
npx skills add agent-rt/skills
```

Pick a single skill:

```sh
npx skills add agent-rt/skills/<skill-name>
```

Install globally (default is project-scoped):

```sh
npx skills add -g agent-rt/skills
```

See `npx skills --help` for the full CLI.

## Install via [skillctl](https://github.com/agent-rt/skillctl)

The same `SKILL.md` files work with `skillctl`'s gateway-only protocol:

```sh
brew tap agent-rt/tap
brew install skillctl

# Add a single skill (skillctl namespace = `<namespace>/<name>`)
skillctl add github:agent-rt/skills/skills/release-flow --as agent-rt/release-flow
skillctl add github:agent-rt/skills/skills/rust-review  --as official/rust-review
skillctl add github:agent-rt/skills/skills/skillctl-self --as local/skillctl-self
```

Then apply a profile to start a project pre-loaded with these skills:

```sh
skillctl profile add github:agent-rt/skills/profiles/skillctl-dev.toml
cd my-project
skillctl init skillctl-dev
```

## Repo layout

```
skills/         agentskills.io-compatible SKILL.md folders
  cmcp/             How to use cmcp (curl for MCP)
  synap/            How to use synap (structured agent memory engine)
  rust-review/      Idiomatic Rust review checklist
  skillctl-self/    Conventions for working ON skillctl itself
  release-flow/     Agent-RT release pipeline + family conventions
profiles/       skillctl-only profiles (TOML)
  rust-cli.toml         Rust CLI starter (core: rust-review)
  skillctl-dev.toml     For developing skillctl itself
template/       Reference SKILL.md template
.claude-plugin/ Claude plugin marketplace metadata
```

## Naming convention

Skill folder names match `frontmatter.name` (agentskills.io strict rule).
For skillctl users, the namespaced ID is set per-skill via `--as <namespace>/<name>`
on `skillctl add`, or via the recommended IDs documented next to each skill above.

## Contributing

PRs welcome. Each skill must:

- Live in `skills/<name>/SKILL.md`
- Have frontmatter `name: <same as folder>`, `description:` (used for triggering),
  and follow the [agentskills.io spec](https://agentskills.io)
- Pass `npx skills validate skills/<name>` (when that command exists)

Optional skillctl extras (ignored by agentskills.io):

- `metadata.namespace`, `metadata.triggers`, `metadata.languages`,
  `metadata.tags`, `metadata.requires`

## License

Apache-2.0
