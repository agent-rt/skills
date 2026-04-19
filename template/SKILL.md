---
name: my-skill
description: One sentence describing what this skill does AND the specific contexts when the agent should use it. Descriptions that only say "what" get under-triggered — include "when". Example: "How to render dependency graphs as SVG. Use this whenever the user mentions dependencies, call graphs, or asks to visualize relationships between modules."
---

# My Skill

A one-paragraph overview of the skill. Who is it for, what problem does it
solve, what's the headline workflow?

## When to use

- Concrete user phrase or context 1
- Concrete user phrase or context 2
- Concrete user phrase or context 3

## Steps

1. First step — be imperative ("Run …", "Check …"), not descriptive.
2. Second step.
3. Third step.

## Examples

**Input:** <what the user says or provides>
**Output:** <what the agent produces>

## Notes

- If you have bundled scripts, tell the agent to use them instead of
  reinventing helpers. Example: "Run `scripts/render.py <input>` rather
  than writing your own SVG generator."
- If you have large reference files, mention them here with a one-line
  summary so the agent can decide whether to load them.

## Resources

- `scripts/` — executable helpers
- `references/` — longer docs loaded on demand
- `assets/` — templates, fonts, icons used in output
