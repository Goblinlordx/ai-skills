---
name: conductor-parallel
description: Auto-detect worker/coordinator role in a parallel worktree workflow, then enter the appropriate mode (worker implements tracks and merges; coordinator orchestrates serial merges).
metadata:
  argument-hint: "[--worker | --coordinator]"
---

# Conductor Parallel

Automate the parallel worktree development workflow. Detects whether this Claude Code instance is a **worker** (in a worktree) or a **coordinator** (on main), confirms the role, then enters the appropriate mode.

## Use this skill when

- Running parallel development with bare-clone repos and multiple worktrees
- You need a worker to implement tracks, verify, and merge back into main
- You need a coordinator to orchestrate the serial merge protocol across workers

## Do not use this skill when

- Working in a single-branch, single-terminal workflow (use `/conductor-implement` instead)
- The project has no Conductor artifacts set up (use `/conductor-setup` first)

---

## After Compaction

When entering the worker or coordinator role, output this anchor line exactly:

```
ACTIVE ROLE: conductor-parallel {worker|coordinator} — skill at ~/.claude/skills/conductor-parallel/SKILL.md
```

This line is designed to survive compaction summaries. If you see it in your context but can no longer recall the full workflow, re-read the skill file before continuing. For project-specific values, re-read only what you need:

- Verification commands → `.agent/conductor/workflow.md`
- Track list/statuses → `.agent/conductor/tracks.md`
- Track progress → `tracks/{trackId}/plan.md`
- Main worktree path → `git worktree list`

---

## Phase 1: Role Detection

### Step 1 — Detect git context

Run these commands to determine the current environment:

```bash
git rev-parse --git-common-dir 2>/dev/null
git rev-parse --git-dir 2>/dev/null
git branch --show-current 2>/dev/null
```

**Decision logic:**

| `--git-common-dir` vs `--git-dir` | Branch          | Detected role   |
|------------------------------------|-----------------|-----------------|
| Different (worktree)               | Not main/master | **Worker**      |
| Different (worktree)               | main/master     | **Worker** (unusual — flag it) |
| Same (not a worktree)              | main/master     | **Coordinator** |
| Same (not a worktree)              | Not main/master | Ambiguous — ask |

If the argument `--worker` or `--coordinator` was provided, use that directly and skip detection.

### Step 2 — Confirm role with user

Present the detected role and ask for confirmation:

```
Detected environment:
  Git dir:        {git-dir}
  Common dir:     {git-common-dir}
  Branch:         {branch}
  Detected role:  {Worker | Coordinator}

Enter this role?
1. Yes, enter {role} mode
2. No, switch to {other role} mode
```

**CRITICAL: Wait for explicit user confirmation before proceeding.**

---

## Phase 2A: Worker Mode

### Pre-flight Checks

1. Verify Conductor is initialized:
   - Check `.agent/conductor/product.md` exists
   - Check `.agent/conductor/workflow.md` exists
   - Check `.agent/conductor/tracks.md` exists
   - If missing: Display error and suggest running `/conductor-setup` first

2. Load workflow configuration:
   - Read `.agent/conductor/workflow.md`
   - Parse verification commands: `make test` (build + unit tests) and `make e2e` (full-stack integration)

3. Verify worktree setup:
   - Run `git worktree list` to confirm main worktree exists
   - Record the main worktree path for later merge operations
   - Record the current branch as the **worker home branch** (this is where the worker returns between tracks)

### Worker Loop

The worker repeats this cycle: **Select → Branch → Implement → Verify → Pause → Merge → Cleanup → Loop**

---

#### Step W1: Track Selection

1. Read `.agent/conductor/tracks.md`
2. Show available (incomplete) tracks **grouped by type**, with a refresh option and a prompt that accepts typing a track ID directly (since the list may be long):

```
Available tracks:

Features:
  [~] auth_20250115100000Z - User Authentication (Phase 2, Task 3)
  [ ] dashboard_20250113140000Z - Dashboard Feature

Bugs:
  [ ] nav-fix_20250114081500Z - Navigation Bug Fix

Chores:
  [ ] cleanup_20250120160000Z - Remove deprecated endpoints

R. Refresh (reset to main and re-read tracks)

Type a track ID to select, or R to refresh:
```

3. **If the user chooses "R" (refresh):** Reset the worker home branch to main and re-read tracks:
   ```bash
   git reset --hard main
   ```
   This gives the worker a clean slate matching the latest main, picking up any new tracks the coordinator committed. Then re-display the updated list and loop back to the top of Step W1. This is safe because the worker has not started implementation yet.

4. Once a track is selected, load its `metadata.json` to determine the track type (feature, bug, chore, refactor).

#### Step W1b: Create Implementation Branch

Create a dedicated branch for this track based on main:

```bash
git checkout -b {type}/{trackId} main
```

Branch naming convention: `{type}/{trackId}` where type comes from the track's metadata (e.g. `feature/auth_20250115100000Z`, `bug/nav-fix_20250114081500Z`, `chore/cleanup_20250120160000Z`).

Then load the track's context:
   - `.agent/conductor/tracks/{trackId}/spec.md`
   - `.agent/conductor/tracks/{trackId}/plan.md`
   - `.agent/conductor/tracks/{trackId}/metadata.json`
   - `.agent/conductor/product.md`
   - `.agent/conductor/tech-stack.md`
   - `.agent/conductor/code_styleguides/` (if present)

#### Step W2: Implementation

Implement the entire track following the same patterns as `/conductor-implement`:

- Follow TDD workflow if configured in `workflow.md`
- Execute each task in the plan sequentially
- Commit after each task completion using the commit strategy from `workflow.md`
- Update `plan.md` and `metadata.json` as tasks complete
- Follow all rules from `.agent/conductor/workflow.md`

**IMPORTANT:** Implementation is the same as `/conductor-implement` — follow all its task execution, error handling, and progress tracking patterns. The difference is what happens AFTER all tasks are complete (verification and merge instead of cleanup/archive).

#### Step W2b: Mark Track Complete

After all tasks are done, update **all three** tracking files and commit:

1. `.agent/conductor/tracks.md` — change `[ ]` or `[~]` to `[x]` for this track
2. `.agent/conductor/tracks/{trackId}/plan.md` — ensure all tasks are `[x]`
3. `.agent/conductor/tracks/{trackId}/metadata.json` — set `status: "complete"`

```bash
git add .agent/conductor/tracks.md .agent/conductor/tracks/{trackId}/
git commit -m "chore: mark track {trackId} complete"
```

**This is critical.** If `tracks.md` is not updated before merge, the completed track will appear stale/incomplete on main after the merge.

#### Step W3: Automatic Verification

After the track is marked complete, run the **full verification suite** as defined in `.agent/conductor/workflow.md`.

Run the verification commands from `workflow.md`:

```bash
# Step 1: Build + Unit Tests
make test

# Step 2: Full-stack E2E (Docker lifecycle + integration tests)
make e2e
```

Run both steps. Collect results for each.

#### Step W4: Pause — Report and Wait

Present verification results and **STOP**:

```
================================================================================
                    TRACK COMPLETE — VERIFICATION RESULTS
================================================================================
Track:      {trackId} - {title}
Branch:     {worker-branch}
Tasks:      {completed}/{total}

Verification:
  Build + Unit Tests (make test):  {PASS | FAIL: error summary}
  E2E Tests (make e2e):            {PASS | FAIL: error summary}

Overall:    {ALL PASS | FAILURES DETECTED}
================================================================================

{If ALL PASS:}
Ready to merge. Waiting for coordinator to say "merge".

{If FAILURES:}
Verification failures detected. Fix issues before merging.
Options:
1. Attempt to fix failures
2. Show failure details
3. Wait for manual intervention
================================================================================
```

**CRITICAL: Do NOT proceed to merge automatically. Wait for the user (coordinator) to explicitly say "merge" or give a merge command.**

If there are verification failures, help fix them and re-run verification before allowing merge.

#### Step W5: Merge Sequence

When the user says "merge" (or equivalent), execute this exact sequence:

##### 5a. Rebase onto latest main

```bash
git rebase main
```

- If rebase succeeds: continue to 5b
- If rebase conflicts:
  ```bash
  git rebase --abort
  ```
  ```
  REBASE CONFLICT — Cannot auto-merge.

  Conflicting files:
    {list of conflicting files}

  Options:
  1. Show conflict details
  2. Attempt manual resolution
  3. Abort and wait for coordinator
  ```
  **HALT and wait for instructions.** Do NOT force through conflicts.

##### 5b. Re-run full verification

After rebase, main's changes are incorporated. Run the full verification suite again:

```bash
make test
make e2e
```

- If all pass: continue to 5c
- If failures:
  ```
  POST-REBASE VERIFICATION FAILED

  The rebase incorporated changes from main that caused failures:
    {failure details}

  Options:
  1. Attempt to fix
  2. Show failure details
  3. Abort merge and wait for manual intervention
  ```
  **HALT and wait for instructions.**

##### 5c. Fast-forward merge into main

Use `git -C` to operate on the main worktree without changing directories:

```bash
git -C {main-worktree-path} merge {type}/{trackId} --ff-only
```

- If merge succeeds (exit code 0): continue to 5d
- If merge fails (not fast-forwardable):
  ```
  MERGE FAILED — Branch is not fast-forwardable.

  This usually means main has advanced since the rebase.
  Another worker may have merged between rebase and merge.

  Options:
  1. Re-run rebase and try again
  2. Abort and wait for coordinator
  ```
  **HALT and wait for instructions.**

##### 5d. Confirm merge, return to home branch, and clean up

```bash
# Verify merge succeeded by checking main's log
git -C {main-worktree-path} log --oneline -3

# Return to worker home branch
git checkout {worker-home-branch}

# Reset home branch to main (pick up the merge + any other changes)
git reset --hard main

# Delete the implementation branch
git branch -d {type}/{trackId}
```

Report:

```
================================================================================
                         MERGE COMPLETE
================================================================================
Track:       {trackId} - {title}
Merged into: main
Branch:      {type}/{trackId} (deleted)
Home branch: {worker-home-branch} (reset to main)

Main worktree log:
  {last 3 commits}

Worker is ready for next track.
================================================================================
```

#### Step W6: Loop

After successful merge and cleanup:

1. The worker is now on its home branch, reset to main — ask for next track (back to Step W1)
2. If no more tracks: report completion

```
All tracks complete! No remaining work.
Run /conductor-status to see project summary.
```

### Worker Constraints

- **Local branches only** — NEVER run `git push` in worker mode
- **Always pause after verification** — never auto-merge
- **Serial merges** — one merge at a time, coordinated by the user/coordinator
- **One branch per track** — create `{type}/{trackId}` from main, delete after merge
- **Clean up after merge** — return to home branch, reset to main, delete implementation branch
- **Verification is mandatory** — both before and after rebase

---

## Phase 2B: Coordinator Mode

### Pre-flight Checks

1. Verify on main/master branch
2. Verify Conductor is initialized (same checks as worker)
3. Run `git worktree list` to discover active worktrees

### Coordinator Display

Show project status and active worktrees:

```
================================================================================
                    CONDUCTOR — COORDINATOR MODE
================================================================================
Branch:     main
Worktrees:  {count} active

Active worktrees:
  {path1} [{branch1}]
  {path2} [{branch2}]
  ...

Project Status:
  Tracks: {completed}/{total}
  {brief track summary from tracks.md}

================================================================================
                       SERIAL MERGE PROTOCOL
================================================================================

To merge worker results safely:

1. Tell ONE worker to merge (say "merge" in their terminal)
2. Wait for that worker to confirm merge complete
3. Then tell the NEXT worker to merge
4. Repeat until all workers are merged

IMPORTANT: Only one merge at a time! Workers rebase onto main,
so concurrent merges will cause conflicts.

================================================================================
                         AVAILABLE COMMANDS
================================================================================

  /conductor-status      — Full project status
  /conductor-implement   — Implement a track (in this terminal)
  /conductor-new-track   — Create a new track
  /conductor-validate    — Validate project artifacts

================================================================================
```

### Coordinator Ongoing Role

The coordinator stays in an advisory mode:

- Answer questions about project status
- Help triage issues if a worker reports merge conflicts or verification failures
- Remind the serial merge protocol if the user tries to merge multiple workers simultaneously
- Run `/conductor-status` on request to show updated progress after merges
- **Always commit after creating new tracks.** Every time `/conductor-new-track` is run, immediately commit the new track artifacts so workers can see them on refresh:
  ```bash
  git add .agent/conductor/tracks.md .agent/conductor/tracks/{trackId}/
  git commit -m "chore: add track {trackId}"
  ```
  This applies in both coordinator and worker contexts — if a worker creates a track from main before branching, it must be committed too. Workers pick up new tracks via `git reset --hard main` (refresh) or when creating a new implementation branch from main.

### Coordinator Actions

#### Bulk Archive

Archive all completed tracks at once. Invoke via `/conductor-bulk-archive`.

#### Compact Archive

Remove archived track directories from the working tree while preserving recovery via git history with rich metadata tracking. Invoke via `/conductor-compact-archive`.

---

## Error Handling Summary

| Error                      | Worker Action                                           |
|----------------------------|---------------------------------------------------------|
| Conductor not initialized  | Display error, suggest `/conductor-setup`               |
| Track not found            | Show available tracks, ask for correction               |
| Verification failure       | Report details, offer fix/retry/wait                    |
| Rebase conflict            | `git rebase --abort`, report conflicts, halt            |
| Post-rebase verify failure | Report details, offer fix/retry/abort                   |
| Merge not fast-forwardable | Report, offer re-rebase or abort                        |
| Git operation failure      | Show `git status`, report error, halt                   |
| Worktree not found         | Verify `git worktree list`, suggest setup               |

---

## Critical Rules

1. **NEVER push to remote** — all worker branches are local only
2. **NEVER auto-merge** — always wait for explicit "merge" command
3. **ALWAYS verify twice** — before pause AND after rebase
4. **ALWAYS use --ff-only** — no merge commits, clean fast-forward only
5. **ALWAYS reset after merge** — `git reset --hard main` before next track
6. **ONE merge at a time** — serial merge protocol prevents conflicts
7. **Follow workflow.md** — all TDD, commit, and verification rules apply
8. **HALT on any failure** — do not attempt to continue past errors without user input
