# Markdown / Agent Skills Styleguide

This repository primarily manages "Agent Skills", which are standard operating procedures and declarative directives for Gemini Antigravity (and potentially other compatible models).

When creating or modifying a skill, please adhere strictly to the following specification based on [agentskills.io/specification](https://agentskills.io/specification).

## Directory Structure

Each skill should reside in its own directory that matches the skill name exactly.

Required layout:

```
skill-name/
└── SKILL.md # Required
```

Optional layout:

```
skill-name/
├── SKILL.md
├── scripts/    # Helper scripts used by the skill
├── references/ # Reference documentation, examples, etc.
└── assets/     # Images or binary files required
```

## SKILL.md Format

Every skill must have a `SKILL.md` containing a **Frontmatter** and a **BodyContent**.

### Frontmatter (Required)

A valid YAML frontmatter block must start and end with `---`.

```yaml
---
name: skill-name
description: A description of what this skill does and when to use it.
license: Apache-2.0
compatibility: Requires Go and Node.js
metadata:
  version: "1.0"
---
```

**Fields:**

- `name` (Required): Must be 1-64 characters, `a-z` and `-` only. No consecutive hyphens. Cannot start or end with a hyphen. **Must match the parent directory name exactly**.
- `description` (Required): Must be 1-1024 characters. Should describe both what the skill does and when to use it, emphasizing keywords an agent might naturally encounter.
- `license` (Optional): Short string identifying the license.
- `compatibility` (Optional): 1-500 chars. Only include if your skill has specific environment/tooling requirements.
- `metadata` (Optional): Key-value pair strings for additional undocumented properties.
- `allowed-tools` (Optional): Experimental space-delimited list of tools that are pre-approved to run.

### Body Content

The body of `SKILL.md` should be concise, direct documentation for the agent.

Must include:

1. **Step-by-step instructions** on how the skill should be executed.
2. **Examples of inputs and outputs** where helpful (e.g. referencing `scripts/` or code patterns).
3. **Common edge cases** the agent should be aware of.

## Progressive Disclosure & File References

- Keep `SKILL.md` token-efficient (recommended <5000 tokens).
- Reference any other materials stored in `scripts/`, `references/`, or `assets/` so the exact files are only read proactively when needed by the agent. Example: `See [the reference guide](references/REFERENCE.md) for details.`

## Validation

Remember to validate against the official `skills-ref validate ./skill-name` mechanism if available.
