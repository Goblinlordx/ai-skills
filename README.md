# AI Skills

This repository is a collection of AI Skills, which provide custom instructions, workflows, and guidelines for AI agents to follow.

## Contents

### `basic-dev`

The `basic-dev` set of skills is intended to act as global guidance for AI agents performing development work. These guidelines are opinionated and are based on my specific development preferences and standards.

### `conductor`

The `conductor` folder contains a slightly modified version of the Conductor framework designed to be used with Antigravity. **Note:** This has only been tested minimally at the moment.

## Installation & Usage

### Antigravity

**Global Usage**
To make these skills globally available for Antigravity across all your projects, you need to place them in your global Antigravity skills directory.

Copy or symlink the specific skill folders into:

```bash
~/.gemini/antigravity/skills/
```

_(For example, copying the contents of `basic-dev/` into this directory will register them as global skills)._

**Project-Specific Usage**
If you want to scope the skills to an individual project rather than globally, you can place the skill folders within the root of your project repository. Antigravity looks for local skills in any of the following directories:

- `.agents/skills/`
- `.agent/skills/`
- `_agents/skills/`
- `_agent/skills/`

### Other IDEs and Agent Software

For other AI coding assistants, IDEs, or agent frameworks (such as Cursor, GitHub Copilot, Cline, etc.), the required locations and formats for global or workspace-specific rules will vary. You may need to place these guidelines in `.cursorrules`, `.github/copilot-instructions.md`, or other software-specific configuration directories for them to be recognized for global usage. Please consult your specific agent software's documentation for exact paths.
