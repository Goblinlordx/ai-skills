---
name: conductor-developer
description: Receive a track ID, validate it is an active unclaimed track, then implement it following the conductor workflow. Worker role in the track generation => approval => push to worker pipeline.
metadata:
  argument-hint: "<track-id>"
---

# Conductor Developer

Implement a conductor track in a parallel worktree workflow. Receives a track ID, validates it is available for work, then executes the full implementation cycle: branch, implement, verify, and merge.

## Use this skill when

- A track has been generated and approved via `/conductor-track-generator`
- You have been assigned a specific track ID to implement
- You are a developer worker in a parallel worktree setup

## Do not use this skill when

- You need to create new tracks (use `/conductor-track-generator` instead)
- The project has no Conductor artifacts (use `/conductor-setup` first)
- You are working in a single-branch workflow (use `/conductor-implement` instead)

---

## After Compaction

When entering the developer role, output this anchor line exactly:

```
ACTIVE ROLE: conductor-developer — track {trackId} — skill at ~/.claude/skills/conductor-developer/SKILL.md
```

This line is designed to survive compaction summaries. If you see it in your context but can no longer recall the full workflow, re-read the skill file before continuing. For project-specific values, re-read only what you need:

- Verification commands: `.agent/conductor/workflow.md`
- Track list/statuses: `.agent/conductor/tracks.md`
- Track progress: `tracks/{trackId}/plan.md`
- Main worktree path: `git worktree list`

---

## Worktree Convention

This agent is expected to run in a worktree whose folder name starts with `developer-` (e.g., `developer-1`, `developer-2`, `developer-3`). The corresponding branch name matches the folder name.

### Step 0 — Verify worktree identity

```bash
git branch --show-current
git rev-parse --git-common-dir 2>/dev/null
git rev-parse --git-dir 2>/dev/null
git worktree list
```

- The current branch should match `developer-*` (this is the **home branch**)
- If not on a `developer-*` branch, warn but continue
- Record the **main worktree path** from `git worktree list` — needed for merge operations
- Record the **home branch** (the `developer-*` branch) — to return to after merge

**All track state reads should come from main** (via `git show main:<path>`) to see the latest committed state, not the local working tree which may be stale.

---

## Phase 1: Validation

### Step 1 — Parse track ID

If no argument was provided:

```
ERROR: Track ID required.

Usage: /conductor-developer <track-id>

To see available tracks, check .agent/conductor/tracks.md or run /conductor-track-generator to create new ones.
```

**HALT.**

### Step 2 — Verify Conductor is initialized

Check these files exist (read from main):
```bash
git show main:.agent/conductor/product.md > /dev/null 2>&1
git show main:.agent/conductor/workflow.md > /dev/null 2>&1
git show main:.agent/conductor/tracks.md > /dev/null 2>&1
```

If missing: Display error and suggest `/conductor-setup`. **HALT.**

### Step 3 — Validate track exists and is claimable

1. **Check track directory exists on main:**
   ```bash
   git show main:.agent/conductor/tracks/{trackId}/spec.md > /dev/null 2>&1
   ```
   If not found:
   ```
   ERROR: Track not found — {trackId}

   The track .agent/conductor/tracks/{trackId}/ does not exist on main.
   This track ID may be incorrect, or the track may not have been merged to main yet.

   Available tracks (from main):
   {list incomplete tracks from `git show main:.agent/conductor/tracks.md`}
   ```
   **HALT.**

2. **Check track status on main:**
   ```bash
   git show main:.agent/conductor/tracks.md
   ```

   - If track is marked `[x]` (complete):
     ```
     ERROR: Track already complete — {trackId}

     This track has already been implemented and marked complete on main.
     ```
     **HALT.**

   - If track is marked `[~]` (in progress), check if another worker has it:

3. **Check if another worker has claimed it:**
   ```bash
   git worktree list
   git branch --list 'feature/*' 'bug/*' 'chore/*' 'refactor/*'
   ```

   Look for a branch matching `*/{trackId}`. If found:
   ```
   ERROR: Track already claimed — {trackId}

   Branch {type}/{trackId} already exists, indicating another worker is implementing this track.

   Worktree: {worktree path if identifiable}
   Branch:   {branch name}

   Choose a different track or coordinate with the other worker.
   ```
   **HALT.**

4. **Check track has required files on main:**
   ```bash
   git show main:.agent/conductor/tracks/{trackId}/spec.md > /dev/null 2>&1
   git show main:.agent/conductor/tracks/{trackId}/plan.md > /dev/null 2>&1
   ```
   If either is missing:
   ```
   ERROR: Track incomplete — {trackId}

   Missing required files on main:
   - {list missing files}

   This track may need to be regenerated via /conductor-track-generator.
   ```
   **HALT.**

### Step 4 — Confirm and enter developer mode

```
================================================================================
                    CONDUCTOR DEVELOPER — TRACK VALIDATED
================================================================================

Track:    {trackId}
Title:    {title from metadata.json or spec.md}
Type:     {type}
Tasks:    {total tasks from plan.md}
Phases:   {total phases}

Ready to begin implementation. This will:
1. Create branch {type}/{trackId} from main
2. Implement all tasks following the plan
3. Verify and prepare for merge

Proceed?
================================================================================
```

**Wait for user confirmation.**

Output the compaction anchor:
```
ACTIVE ROLE: conductor-developer — track {trackId} — skill at ~/.claude/skills/conductor-developer/SKILL.md
```

---

## Phase 2: Setup

### Step 5 — Sync home branch and create implementation branch

The `developer-*` home branch is a dead/marker branch. Its only purpose is recording the point at which this worker last synced with main. Sync it now so the marker reflects where we're starting from, then branch off main:

```bash
# Sync home branch to main (updates the marker)
git reset --hard main

# Create implementation branch from main
git checkout -b {type}/{trackId} main
```

Branch naming: `{type}/{trackId}` where type comes from metadata (e.g., `feature/auth_20250115100000Z`).

> **Note:** The implementation branch is created from `main`, not from the home branch. The `git reset --hard main` just before serves as a timestamp marker — it records when this worker last synced, which can be useful for diagnosing staleness.

### Step 6 — Load workflow configuration

Read `.agent/conductor/workflow.md` and parse:
- Verification commands (e.g., `make test`, `make e2e`)
- TDD strictness level
- Commit strategy

### Step 7 — Load track context

Read all track files (now from the working tree, which is based on main):
- `.agent/conductor/tracks/{trackId}/spec.md`
- `.agent/conductor/tracks/{trackId}/plan.md`
- `.agent/conductor/tracks/{trackId}/metadata.json`
- `.agent/conductor/product.md`
- `.agent/conductor/tech-stack.md`
- `.agent/conductor/code_styleguides/` (if present)

---

## Phase 3: Implementation

### Step 8 — Execute the plan

Follow the exact same implementation workflow as `/conductor-implement`:

- Execute each task in the plan sequentially
- Follow TDD workflow if configured in `workflow.md`
- Commit after each task completion using the commit strategy from `workflow.md`
- Update `plan.md` task markers: `[ ]` -> `[~]` -> `[x]`
- Update `metadata.json` as tasks complete
- Run phase verification at the end of each phase
- **Wait for user approval between phases**

### Step 9 — Mark track complete

After all tasks are done, update all tracking files and commit:

1. `.agent/conductor/tracks.md` — change `[ ]` or `[~]` to `[x]`
2. `.agent/conductor/tracks/{trackId}/plan.md` — ensure all tasks `[x]`
3. `.agent/conductor/tracks/{trackId}/metadata.json` — set `status: "complete"`

```bash
git add .agent/conductor/tracks.md .agent/conductor/tracks/{trackId}/
git commit -m "chore: mark track {trackId} complete"
```

---

## Phase 4: Merge

### Step 10 — Report completion and wait

```
================================================================================
                    TRACK COMPLETE — READY TO MERGE
================================================================================
Track:      {trackId} - {title}
Branch:     {type}/{trackId}
Tasks:      {completed}/{total}

Ready to merge. Say "merge" to begin the lock -> rebase -> verify -> merge sequence.
================================================================================
```

**CRITICAL: Do NOT proceed to merge automatically. Wait for explicit "merge" command.**

### Step 11 — Merge sequence

When the user says "merge", execute the full merge protocol:

#### 11a. Acquire merge lock

```bash
LOCK_DIR="$(git rev-parse --git-common-dir)/merge.lock"
if ! mkdir "$LOCK_DIR" 2>/dev/null; then
  echo "MERGE LOCK HELD — Another worker is currently merging. Wait for them to finish."
  exit 1
fi
echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) $(git branch --show-current)" > "$LOCK_DIR/info"
echo "Merge lock acquired"
```

If lock held: report and **HALT**.

**From this point: release lock on ANY failure:**
```bash
LOCK_DIR="$(git rev-parse --git-common-dir)/merge.lock" && rm -rf "$LOCK_DIR"
```

#### 11b. Rebase onto latest main

```bash
LOCK_DIR="$(git rev-parse --git-common-dir)/merge.lock"
if ! git rebase main; then
  git rebase --abort 2>/dev/null || true
  rm -rf "$LOCK_DIR"
  echo "REBASE FAILED — lock released"
  exit 1
fi
echo "Rebase succeeded"
```

On conflict: lock released automatically. Report and **HALT**.

#### 11c. Post-rebase verification

Run the full verification suite from `workflow.md` (e.g., `make test`, `make e2e`).

On failure: release lock, report, **HALT**.

#### 11d. Fast-forward merge into main

```bash
LOCK_DIR="$(git rev-parse --git-common-dir)/merge.lock"
if git -C {main-worktree-path} merge {type}/{trackId} --ff-only; then
  rm -rf "$LOCK_DIR"
  echo "MERGE SUCCEEDED — lock released"
else
  rm -rf "$LOCK_DIR"
  echo "MERGE FAILED — lock released"
  exit 1
fi
```

On failure: lock released. Report and **HALT**.

#### 11e. Cleanup — return to home branch

```bash
# Verify merge
git -C {main-worktree-path} log --oneline -3

# Delete implementation branch (safe — it's been merged)
git branch -d {type}/{trackId}

# Return to developer home branch
git checkout {developer-home-branch}

# Sync home branch to main (updates the marker to post-merge state)
git reset --hard main
```

Report:

```
================================================================================
                         MERGE COMPLETE
================================================================================
Track:       {trackId} - {title}
Merged into: main
Branch:      {type}/{trackId} (deleted)
Home branch: {developer-home-branch} (synced to main)

Developer is ready for next track.
================================================================================
```

---

## Error Handling Summary

| Error                      | Action                                                   |
|----------------------------|----------------------------------------------------------|
| No track ID provided       | Display usage, **HALT**                                  |
| Track not found on main    | List available tracks from main, **HALT**                |
| Track already complete     | Notify, **HALT**                                         |
| Track already claimed      | Show claiming worker/branch, **HALT**                    |
| Track missing spec/plan    | Suggest regeneration, **HALT**                           |
| Conductor not initialized  | Suggest `/conductor-setup`, **HALT**                     |
| Verification failure       | Report details, offer fix/retry/wait                     |
| Merge lock held            | Report, wait for other worker                            |
| Rebase conflict            | Abort rebase, release lock, report, **HALT**             |
| Post-rebase verify failure | Release lock, report, offer fix/retry/abort              |
| Merge not fast-forwardable | Release lock, offer re-rebase or abort                   |

---

## Critical Rules

1. **ALWAYS validate before implementing** — never start work on an invalid or claimed track
2. **ALWAYS read track state from main** — use `git show main:<path>`, not local working tree
3. **NEVER push to remote** — all worker branches are local only
4. **NEVER auto-merge** — always wait for explicit "merge" command
5. **ALWAYS verify after rebase** — full verification after rebase, before merge
6. **ALWAYS use --ff-only** — clean fast-forward merges only
7. **ONE merge at a time** — enforce via cross-worktree merge lock
8. **HALT on any failure** — do not continue past errors without user input
9. **Follow workflow.md** — all TDD, commit, and verification rules apply
10. **Return to home branch** — always checkout back to `developer-*` branch after merge
