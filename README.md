# agent-rt/skills

Agent Skills curated by [agent-rt](https://github.com/agent-rt).

A Skill is a folder with a `SKILL.md` file — YAML frontmatter + Markdown
instructions — that an AI agent loads on demand to handle a specific task
well. See [agentskills.io](https://agentskills.io) for the spec.

## Install

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

## Layout

```
skills/
  <skill-name>/
    SKILL.md        # required: YAML frontmatter + instructions
    scripts/        # optional: executable helpers
    references/     # optional: docs loaded on demand
    assets/         # optional: templates, icons, fonts

template/SKILL.md   # starter template for new skills
.claude-plugin/     # Claude Code plugin-marketplace manifests
```

## Contributing a skill

1. Copy `template/` to `skills/<your-skill-name>/`.
2. Edit `SKILL.md` — fill in `name`, `description`, and the instruction body.
3. Add any `scripts/` / `references/` / `assets/` the skill needs.
4. Open a PR.

Skill description quality is the primary trigger for whether an agent picks
up the skill — spend time on it. Describe *when* to use the skill, not just
*what* it does.

## License

Apache-2.0 unless noted otherwise inside an individual skill folder.
