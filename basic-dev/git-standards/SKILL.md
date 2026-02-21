---
name: git-standards
description: Enforce standard Git workflows, conventional commits, and trunk-based development practices for AI agents.
---

# Git Standards and Workflow

This skill outlines the mandatory Git practices, commit formatting, and development workflow that AI agents must follow when collaborating on codebases.

## 1. Conventional Commits

All commit messages MUST adhere to the [Conventional Commits](https://www.conventionalcommits.org/) specification. This ensures a readable history and facilitates automated versioning and changelog generation.

### Format

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Types

- **feat**: A new feature
- **fix**: A bug fix
- **docs**: Documentation only changes
- **style**: Changes that do not affect the meaning of the code (white-space, formatting, missing semi-colons, etc)
- **refactor**: A code change that neither fixes a bug nor adds a feature
- **perf**: A code change that improves performance
- **test**: Adding missing tests or correcting existing tests
- **build**: Changes that affect the build system or external dependencies
- **ci**: Changes to our CI configuration files and scripts
- **chore**: Other changes that don't modify project specifications or src
- **revert**: Reverts a previous commit

### Guidelines

- Use the imperative, present tense in the description: "add" not "added" nor "adds".
- Do not capitalize the first letter of the description.
- Do not end the description with a period (.).

## 2. Trunk-Based Development Workflow

By default, development should follow a Trunk-Based Development workflow. The primary active branch (usually `main` or `master`) acts as the trunk.

### Standard Workflow vs. Simple Flow

**Agent Instruction**: Ask the user which workflow they prefer when starting development, particularly for smaller personal projects where a simpler flow might be desired.

1. **Simple Flow (Small/Personal Projects)**: If the user indicates that collaboration is unnecessary or the project is small, you may develop and commit directly to the `main` or `master` trunk branch.
2. **Standard Collaborative Flow (Default)**: Once collaboration is necessary (or when requested by the user), strictly adhere to the feature branch and merge flow:
   - **Branch Creation**: Create a new feature or fix branch from the trunk for the work to be done (e.g., `git checkout -b feat/user-authentication`).
   - **Implementation**: Do the work and commit it to the local feature branch following the Conventional Commits standard.
   - **Pushing to Remote**: The feature branch is pushed to the remote repository to open a Pull Request (PR) or Merge Request (MR).
   - **Review & Merge**: After the PR/MR is fully reviewed and approved, the feature branch is merged back into the trunk.

## 3. Remote Push Policy for AI Agents

**CRITICAL RULE**: An AI agent MUST NEVER push code to a remote repository automatically or without the user's explicit, unambiguous consent.

### Agent Instructions:

- **Never Auto-Push:** Do not include `git push` commands in automated scripts or execute them without pausing for confirmation.
- **Instruct the User:** When local commits are complete and a branch is ready for a PR/MR, you should state that the local work is complete and provide the exact push command for the user.
- **Example Response:**
  > _"I have completed the work and committed it locally to the `feat/user-auth` branch. To proceed, please push this branch to the remote to open a PR by running: `git push -u origin feat/user-auth`. Let me know if you would like me to run this command for you."_

+## 4. Quality Assurance Before Commit

- +Maintaining a healthy and buildable trunk is critical.
- +**CRITICAL RULE**: Code MUST NEVER be committed locally unless all of the following checks pass:
  +- **Linting**: No linting errors or warnings (per project configuration).
  +- **Typechecking**: No static type errors (e.g., TypeScript, Go, Python type hints).
  +- **Build**: The project builds successfully without errors.
  +- **Tests**: All unit and integration tests must pass.
- +**Exception**: You may ONLY bypass this rule if the user EXPLICITLY requests a commit despite one or more of these checks failing. In all other cases, you must fix the issues before committing.
