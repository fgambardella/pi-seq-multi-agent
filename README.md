# Sequential Pi Multi-Agent Workflow

This boilerplate configures a sequential parent-child workflow for [Pi](https://github.com/earendil-works/pi):

- The parent **Architect** agent plans the work, maintains project state, creates implementation branches, delegates one focused task at a time, reviews commits, and merges approved work.
- The child **Implementer** agent works in a separate Pi process, changes production code and tests on the assigned branch, runs the required checks, and returns a commit-based handoff.

Only one agent performs implementation work at a time. This makes the workflow suitable for local models and machines that cannot run multiple inference-heavy agents concurrently. It also keeps implementation details out of the Architect's conversation history while giving each Implementer a narrow, fresh context.

A commit is a **reviewable checkpoint**, not an approval. The Implementer creates commits; the Architect verifies them and is the only agent allowed to merge into `main`.

## How the workflow is isolated

The parent and child run in separate Pi processes, so they do not share conversation histories. Their durable shared state is the repository: `DESIGN.md`, `TASKS.md`, the assigned implementation branch, committed code, and test results included in handoffs.

The directory structure does **not** provide hard instruction isolation. Pi loads project context files by walking upward from the current working directory. Therefore:

- A Pi session started at the project root loads the root `AGENTS.md`.
- A child started from `src` loads both the root `AGENTS.md` and `src/AGENTS.md`.

The two files in this boilerplate contain explicit role-scope rules to handle that inheritance. When running from `src`, the nearer `src/AGENTS.md` establishes the Code Implementer role, and Architect-only Git powers from the inherited root file do not apply.

Pi may also load user-level or higher-level context files. Review those files if an agent behaves contrary to this boilerplate's rules. See Pi's current [project context loader](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/resource-loader.ts) for the discovery behavior.

## Core safety invariants

1. `main` is the integration branch.
2. The Architect creates each implementation branch from `main`.
3. The Implementer never commits directly to `main`.
4. The Implementer commits completed work and stabilized partial work on the assigned branch.
5. The Architect never authors or commits production code or tests.
6. The Architect independently reviews and tests the committed changes.
7. Failed, incomplete, or timed-out work remains on the implementation branch until corrected and approved.
8. Only the Architect may merge an approved branch into `main`.
9. Neither agent pushes to a remote unless the user explicitly requests it.

## Prerequisites

- Git, with a repository whose integration branch is named `main`
- A current Pi installation with a configured model provider
- A shell environment in which the Architect can launch `pi` from `src`
- Enough model context for the project instructions, delegated prompt, relevant source files, and test output

This workflow uses Pi's built-in Bash timeout parameter. No timeout extension or shell wrapper is required. The Architect requests a 1,200-second timeout when launching the child, while the child begins wrapping up after approximately 1,000 seconds. The remaining time is reserved for stabilization, testing, committing, and reporting. See Pi's current [Bash tool implementation](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/bash.ts) for timeout behavior.

## Project Structure

```text
my-pi-project/
├── AGENTS.md          # Architect role and orchestration rules
├── DESIGN.md          # Architectural source of truth; owned by the Architect
├── TASKS.md           # Ordered task backlog and delegated prompts
└── src/
    ├── AGENTS.md      # Implementer role and execution rules
    └── ...            # Production code and tests
```

The boilerplate initially may not contain `DESIGN.md`. When it is missing, the Architect must create and populate it before delegating implementation. Do not create an empty `DESIGN.md`, because an existing but blank file does not provide useful architectural guidance.

For a new project, copy the two authoritative instruction files from this boilerplate instead of reproducing their contents in another document:

```bash
mkdir -p my-pi-project/src
cp /path/to/pi-seq-multi-agent/AGENTS.md my-pi-project/AGENTS.md
cp /path/to/pi-seq-multi-agent/src/AGENTS.md my-pi-project/src/AGENTS.md
touch my-pi-project/TASKS.md
cd my-pi-project
git init -b main
git add AGENTS.md TASKS.md src/AGENTS.md
git commit -m "Initialize sequential agent workflow."
```

If you add the workflow to an existing repository, preserve the existing source tree and copy only the two `AGENTS.md` files to their corresponding locations. Confirm that the repository has a `main` branch before launching the Architect.

## Configuration Files

### Root `AGENTS.md`: Architect contract

The root file is the authoritative configuration for the parent agent. It requires the Architect to:

- Read the project overview, task backlog, and architectural state.
- Create and continuously maintain `DESIGN.md`.
- Discover or choose the project's testing framework.
- Decompose the macro-goal into tasks that an Implementer can complete within 20 minutes.
- Include acceptance criteria, required tests, and exact test commands in every delegated prompt.
- Create a dedicated branch named `implementer/<task-id>-<short-slug>` from `main`.
- Pass the exact branch name to the child and never launch the child while `main` is checked out.
- Inspect the returned commit and the complete branch delta against `main`.
- Run tests independently before accepting the handoff.
- Keep failed or partial work on the implementation branch for follow-up agents.
- Merge only complete, verified work into `main`.

The Architect owns `DESIGN.md` and `TASKS.md`. It may commit changes to those state files separately, but it must not write, stage, or commit production code or tests.

### `src/AGENTS.md`: Implementer contract

The nested file is the authoritative configuration for child sessions started from `src`. It requires the Implementer to:

- Treat the inherited Architect identity as out of scope.
- Read `../DESIGN.md` without modifying it.
- Verify that the exact assigned implementation branch is checked out before editing.
- Refuse to work on `main`, on the wrong branch, or in a working tree containing unexpected changes.
- Change only the delegated production-code and test files.
- Run the exact test commands from the delegated prompt.
- Stage files by explicit path and commit completed or stabilized partial work.
- Return the branch name, commit hash, test results, remaining work, and final working-tree status.
- Avoid all branch creation, switching, merging, rebasing, deletion, and remote operations.

Do not copy abbreviated versions of these contracts into this README. Keeping a single authoritative copy of each role prevents the documentation and executable instructions from drifting apart.

## Initialize `TASKS.md`

You can leave `TASKS.md` empty and ask the Architect to populate it, or seed it with a macro-goal:

```markdown
# Macro-Task: Build a Python user-authentication API

**Testing Framework:** To be determined by the Architect

## Tasks

- [ ] Task 001: Define the user model and validation rules
  - Acceptance criteria:
    - The model validates required fields.
    - Invalid email addresses are rejected.
  - Required tests:
    - Unit tests for valid and invalid users.
  - Test command: `python -m pytest tests/test_user.py`
  - Delegated prompt: To be completed by the Architect.
```

Keep tasks small enough to complete, test, stabilize, and commit within the 20-minute child execution window. The Architect appends new tasks rather than replacing historical entries.

## End-to-End Execution Flow

### 1. Start the Architect

From the project root:

```bash
pi
```

Ask it to begin the macro-task recorded in `TASKS.md`, for example:

```text
Begin working on the macro-task in TASKS.md. Follow the Architect workflow in AGENTS.md.
```

### 2. Plan and prepare repository state

Before delegation, the Architect:

1. Reads `README.md`, `TASKS.md`, and `DESIGN.md` when present.
2. Creates `DESIGN.md` if it is missing.
3. Defines the next focused task, its acceptance criteria, required tests, and exact test commands.
4. Ensures Architect-owned planning changes are committed separately and the working tree is clean.
5. Starts from `main` and creates a branch such as:

```bash
git switch main
git switch -c implementer/001-user-model
```

The Architect must not discard, reset, overwrite, or stash unrelated changes to obtain a clean tree. It stops and reports the conflict instead.

### 3. Delegate one task

The Architect launches the child from `src` with a Bash tool timeout of 1,200 seconds:

```bash
cd src && pi -p "<focused prompt from TASKS.md>"
```

The prompt must identify:

- The exact implementation branch
- `main` as the base and integration branch
- The allowed file and behavior scope
- Acceptance criteria
- Tests to write or update
- Exact test commands
- The requirement to commit successful or stabilized partial work
- Prohibited Git operations

### 4. Perform the Implementer preflight

Before changing files, the child checks:

```bash
git branch --show-current
git status --short
```

The branch must exactly match the assigned branch and must not be `main`. Unexpected pre-existing changes are a blocking failure; the child reports them without cleaning, resetting, stashing, or modifying the repository.

### 5. Implement, test, and commit

The Implementer reads `../DESIGN.md`, makes only the delegated changes, writes the requested tests, and runs the exact test commands. It stages only files it intentionally changed:

```bash
git add path/to/changed-file path/to/test-file
git commit -m "feat: implement focused capability"
git rev-parse HEAD
git status --short
```

Broad staging commands such as `git add .` and `git add -A` are forbidden because they can capture unrelated or Architect-owned changes.

### 6. Return a commit-based handoff

A successful handoff should resemble:

```text
BRANCH: implementer/001-user-model
COMMIT: a1b2c3d

Completed:
- Added the user model and validation.
- Added unit tests for valid and invalid inputs.

Tests:
- python -m pytest tests/test_user.py — PASS

Remaining work:
- None.

Working tree:
- Clean.

RESULT: SUCCESS
Implemented and verified the delegated user-model task.
```

`RESULT: SUCCESS` means the delegated acceptance criteria are complete and every required test passed. A commit does not by itself make the work successful or approved.

### 7. Review the committed work

The Architect verifies the exact checkpoint rather than trusting the summary:

```bash
git branch --show-current
git show <commit-id>
git diff main...<implementation-branch>
```

It confirms that:

- The commit exists and belongs to the assigned branch.
- Only delegated production code and tests changed.
- `AGENTS.md`, `DESIGN.md`, `TASKS.md`, and unrelated files were not changed by the Implementer.
- The implementation follows `DESIGN.md` and the acceptance criteria.
- The required tests pass when run independently.

If review fails, the Architect records the findings and delegates a corrective task on the same branch. It does not merge the branch.

### 8. Update state and merge approved work

After successful review, the Architect updates `DESIGN.md` and `TASKS.md`, commits those Architect-owned changes separately, and merges the completed branch into `main` with a non-fast-forward merge:

```bash
git switch main
git merge --no-ff implementer/001-user-model
```

The Architect runs the relevant tests again after the merge. It deletes the implementation branch only after the merge and post-merge tests succeed. It does not push `main` unless the user explicitly requests it.

## Timeout and partial-work recovery

The child has a strict 1,200-second execution window. After approximately 1,000 seconds, it must stop starting new work and enter Wrap-Up mode:

1. Stabilize syntax and leave the code in the safest practical state.
2. Run the most relevant tests that fit within the remaining time.
3. Stage only delegated files.
4. Create a clearly identified WIP checkpoint commit.
5. Return the commit hash, test status, unfinished work, blockers, and working-tree status.
6. End with `RESULT: FAILURE`, because the delegated task is incomplete.

Example partial handoff:

```text
BRANCH: implementer/002-token-service
COMMIT: d4e5f6a

Completed:
- Added token generation and passing unit tests.

In progress:
- Token refresh validation is stubbed with a TODO.

Tests:
- python -m pytest tests/test_tokens.py — FAIL (2 refresh tests)

Remaining work:
- Implement refresh expiry validation.

Working tree:
- Clean.

RESULT: FAILURE
Committed a stable WIP checkpoint; refresh validation remains incomplete.
```

The Architect reviews this checkpoint but does not merge it merely because it is committed. It decomposes the remaining work and starts another child sequentially on the **same implementation branch**, using the checkpoint commit as the starting state.

If the child made no repository changes, it should not fabricate an empty commit. It reports `COMMIT: NONE` and explains the blocker.

## Troubleshooting

### The child tries to act as the Architect

Confirm that:

- The child was launched from `src`.
- `src/AGENTS.md` exists and contains the Code Implementer role-scope rule.
- No user-level or ancestor context file overrides the intended behavior.
- The delegated prompt does not assign Architect responsibilities to the child.

Remember that the child receives both project `AGENTS.md` files; the nested file narrows the role but does not prevent the root file from entering the model context.

### The Architect tells the child not to commit

That instruction violates this workflow. The Implementer owns implementation commits, including stabilized timeout checkpoints. The Architect owns review and integration. Restate this invariant positively in the delegated prompt and inspect user-level Pi context for competing Git rules.

### The child starts on `main` or the wrong branch

The child must stop before editing and return `RESULT: FAILURE`. The Architect should restore a clean, known repository state without discarding unrelated work, create or check out the intended implementation branch, and delegate again.

### The working tree is unexpectedly dirty

Neither agent should automatically clean, reset, or stash the tree. Determine who owns each change first. Commit Architect-owned state separately, preserve unrelated user work, and delegate only from a clean implementation branch.

### Tests pass in the child but fail after merge

The Architect must stop and report the post-merge failure without pushing. Investigate integration differences on `main`; do not conceal the failure or treat the prior child result as sufficient verification.

## Design Trade-offs

This workflow favors traceability and low concurrent resource use over speed:

- Tasks run sequentially rather than in parallel.
- Each child starts with a fresh conversation and must rediscover some local context.
- Ancestor `AGENTS.md` loading consumes part of the child's context window.
- A shared working tree means branch switches affect both parent and child processes; the Architect must wait for the child to finish before switching branches.
- Dedicated branches, commit hashes, and independent review add Git overhead but provide clear provenance and reliable timeout recovery.

For parallel agents, use separate Git worktrees or separate repository clones. Do not run concurrent Implementers in this shared-worktree design.

## References

- [Pi repository and documentation](https://github.com/earendil-works/pi)
- [Pi project-context loading](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/resource-loader.ts)
- [Pi Bash timeout implementation](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/bash.ts)
