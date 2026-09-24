# Sequential Pi Multi-Agent Workflow

This boilerplate configures a sequential parent-child workflow for [Pi](https://github.com/earendil-works/pi):

- The parent **Architect** plans the work, owns the architecture and task state, creates implementation branches, delegates one focused task at a time, reviews commits, and merges approved work.
- The child **Implementer** runs in a separate Pi process, changes code and tests in its own workspace, verifies the result, commits it on the assigned branch, and returns a commit-based handoff.

Only one agent performs implementation work at a time. This makes the workflow suitable for local models and machines that cannot run multiple inference-heavy agents concurrently. It also keeps implementation details out of the Architect's conversation history while giving each Implementer a narrow, fresh context.

A commit is a **reviewable checkpoint**, not an approval. The Implementer creates implementation commits; the Architect verifies them and is the only agent allowed to merge into `main`.

## Directory and instruction isolation

The Architect and Implementer live in sibling directories:

```text
project-root/
├── architect/
│   └── AGENTS.md
└── implementer/
    └── AGENTS.md
```

Pi discovers project context files by walking upward from the session's working directory. It does not automatically scan sibling directories. Therefore:

- A Pi session started from `architect/` loads `architect/AGENTS.md`, but not `implementer/AGENTS.md`.
- A Pi session started from `implementer/` loads `implementer/AGENTS.md`, but not `architect/AGENTS.md`.

This prevents the Architect's identity and Git powers from leaking into the Implementer's project instructions. It is stronger and simpler than placing the Implementer's file below the Architect's file in one directory hierarchy.

The repository root intentionally has no `AGENTS.md`. If you add one, both agents will inherit it, so it must contain only neutral rules that genuinely apply to both roles. Pi may also load user-level or higher-level context files; inspect those files if either agent behaves contrary to this workflow.

The processes remain isolated at the conversation level, but they share one filesystem and Git working tree. Their durable coordination state consists of:

- `architect/DESIGN.md`
- `architect/TASKS.md`
- The assigned implementation branch
- Implementer commits
- Test results and handoff reports

See Pi's current [project context loader](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/resource-loader.ts) for its context-file discovery behavior.

## Project structure

```text
my-pi-project/
├── .git/                       # One shared Git repository
├── README.md                   # Workflow documentation
├── architect/
│   ├── AGENTS.md               # Architect role and orchestration contract
│   ├── DESIGN.md               # Generated and maintained by the Architect
│   └── TASKS.md                # Macro-goal, task history, and delegated prompts
└── implementer/
    ├── AGENTS.md               # Implementer role and execution contract
    ├── package.json            # Example project manifest; language-dependent
    ├── src/                    # Production source code
    │   └── .gitkeep            # Placeholder; remove after adding source files
    └── tests/                  # Example test location; framework-dependent
```

All implementation artifacts belong under `implementer/`, including source code, tests, dependency manifests, build files, linters, formatters, migrations, and generated project configuration. Harness state belongs under `architect/`.

This boilerplate does not include `architect/DESIGN.md`. The Architect creates and populates it before the first delegation. Do not add an empty `DESIGN.md`: an existing but blank file provides no architectural guidance.

The example manifest and `tests/` directory are illustrative rather than mandatory. Use the conventions of the chosen language and framework, but keep them inside `implementer/`.

## Core safety invariants

1. `main` is the integration branch.
2. The Architect creates each implementation branch from `main`.
3. The Implementer never commits directly to `main`.
4. The Implementer changes repository content only under `implementer/` and never modifies `implementer/AGENTS.md`.
5. The Implementer commits completed work and stabilized partial work on the assigned branch.
6. The Architect changes only state files under `architect/`; it never authors or commits implementation files.
7. The Architect independently reviews and tests the committed changes.
8. Failed, incomplete, or timed-out work remains on its implementation branch until corrected and approved.
9. Only the Architect may merge an approved branch into `main`.
10. Neither agent pushes to a remote unless the user explicitly requests it.

## Prerequisites

- Git, with a repository whose integration branch is named `main`
- A current Pi installation with a configured model provider
- A shell environment in which the Architect can launch a second Pi process from `../implementer/`
- Enough model context for the applicable agent instructions, delegated prompt, relevant project files, and test output

This workflow uses Pi's built-in Bash timeout parameter. No timeout extension or shell wrapper is required. The Architect gives the child-launch Bash call a 1,200-second timeout, while the Implementer begins wrapping up after approximately 1,000 seconds. The remaining time is reserved for stabilization, testing, committing, and reporting.

The timeout is a Bash **tool-call parameter**, not a `pi` command-line option. Conceptually, the Architect invokes:

```text
Bash tool call
├── command: cd ../implementer && pi -p "..."
└── timeout: 1200 seconds
```

See Pi's current [Bash tool implementation](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/bash.ts) for its timeout behavior.

## Set up a new project

Create the sibling workspaces and copy the authoritative agent contracts from this boilerplate:

```bash
mkdir -p my-pi-project/architect my-pi-project/implementer/src
cp /path/to/pi-seq-multi-agent/README.md my-pi-project/README.md
cp /path/to/pi-seq-multi-agent/architect/AGENTS.md my-pi-project/architect/AGENTS.md
cp /path/to/pi-seq-multi-agent/architect/TASKS.md my-pi-project/architect/TASKS.md
cp /path/to/pi-seq-multi-agent/implementer/AGENTS.md my-pi-project/implementer/AGENTS.md
touch my-pi-project/implementer/src/.gitkeep
cd my-pi-project
git init -b main
git add README.md architect implementer
git commit -m "chore: initialize sequential agent workflow"
```

If you add this workflow to an existing repository:

1. Create `architect/` and `implementer/` at the repository root.
2. Move all production code, tests, manifests, and project tooling under `implementer/`.
3. Copy the two `AGENTS.md` files to their corresponding sibling directories.
4. Place `TASKS.md` under `architect/`.
5. Do not leave an Architect `AGENTS.md` at the repository root.
6. Confirm that the integration branch is named `main` and commit the migration before launching either agent.

## Authoritative role contracts

### `architect/AGENTS.md`

Start the parent Pi process from `architect/`. Its contract requires the Architect to:

- Read `../README.md`, local `TASKS.md`, and local `DESIGN.md` when present.
- Inspect only relevant files in `../implementer/` while planning.
- Create and continuously maintain `DESIGN.md`.
- Discover or choose the testing framework used in the implementation workspace.
- Decompose the macro-goal into tasks that fit within the 20-minute child window.
- Add acceptance criteria, required tests, exact test commands, and an implementation branch to every delegated prompt.
- Commit Architect-owned planning state on `main` before creating the implementation branch.
- Create a branch named `implementer/<task-id>-<short-slug>` from `main`.
- Launch the child from `../implementer/` and wait for it to finish.
- Inspect the returned commit and the complete branch delta against `main`.
- Verify that the child changed only allowed files under `implementer/`.
- Run tests independently from `../implementer/`.
- Keep failed or partial work on the same branch for follow-up agents.
- Merge only complete, independently verified work into `main`.

The Architect's direct commits may contain `architect/DESIGN.md` and `architect/TASKS.md`, but it must not write, edit, or stage anything under `implementer/`. The final non-fast-forward merge commit is an integration record that incorporates already reviewed Implementer commits; it is not an Architect-authored implementation change.

### `implementer/AGENTS.md`

Start every child Pi process from `implementer/`. Its contract requires the Implementer to:

- Work only within the implementation workspace.
- Read `../architect/DESIGN.md` without modifying any file under `../architect/`.
- Leave root-level repository files unchanged.
- Verify the exact assigned branch and clean working tree before editing.
- Refuse to work on `main`, on the wrong branch, or with unexpected pre-existing changes.
- Change only files required by the delegated task.
- Run the exact test commands from the prompt.
- Stage files by explicit path and commit completed or stabilized partial work.
- Return the branch, commit hash, test results, remaining work, blockers, and final working-tree status.
- Avoid branch creation, switching, merging, rebasing, deletion, destructive cleanup, and all remote operations.

Do not duplicate shortened copies of these contracts elsewhere. Keeping one authoritative file per role prevents documentation and executable instructions from drifting apart.

## Initialize `architect/TASKS.md`

You may leave `architect/TASKS.md` empty and ask the Architect to populate it, or seed it with a macro-goal:

```markdown
# Macro-Task: Build a Python user-authentication API

**Testing Framework:** To be determined by the Architect

## Tasks

- [ ] Task 001: Define the user model and validation rules
  - Branch: `implementer/001-user-model`
  - Acceptance criteria:
    - The model validates required fields.
    - Invalid email addresses are rejected.
  - Required tests:
    - Unit tests for valid and invalid users.
  - Test command: `python -m pytest tests/test_user.py`
  - Delegated prompt: To be completed by the Architect.
```

The Architect appends tasks rather than deleting history. Each task must be small enough to implement, test, stabilize, commit, and report within 20 minutes.

## End-to-end execution flow

### 1. Start the Architect

Start Pi from the Architect workspace, not from the repository root:

```bash
cd my-pi-project/architect
pi
```

Ask it to begin the macro-goal recorded in its local task tracker:

```text
Begin working on the macro-task in TASKS.md. Follow AGENTS.md and use ../implementer as the implementation workspace.
```

Because `architect/` and `implementer/` are siblings, this session does not load the Implementer's `AGENTS.md`.

### 2. Plan and commit Architect state

Before delegation, the Architect:

1. Reads `../README.md`, `TASKS.md`, and `DESIGN.md` when present.
2. Creates and populates `DESIGN.md` if it is missing.
3. Chooses the exact implementation branch name.
4. Defines a focused task, acceptance criteria, required tests, exact test commands, and delegated prompt.
5. Confirms that `main` is checked out and inspects `git status --short`.
6. Commits its finalized state before creating the implementation branch:

```bash
git add DESIGN.md TASKS.md
git commit -m "docs: plan user-model task"
git switch -c implementer/001-user-model
```

Git discovers the repository root even though these commands run from `architect/`. The Architect must never discard, reset, clean, overwrite, or stash unrelated changes to obtain a clean tree; it stops and reports the conflict instead.

### 3. Delegate one task

The Architect launches the child from the sibling workspace with a Bash tool timeout of 1,200 seconds:

```bash
cd ../implementer && pi -p "<focused prompt from TASKS.md>"
```

The delegated prompt must specify:

- The exact implementation branch
- `main` as the base and integration branch
- Allowed files and behavioral scope under `implementer/`
- Acceptance criteria
- Tests to create or update
- Exact test commands to run from `implementer/`
- The requirement to commit successful or stabilized partial work
- The required handoff fields
- Prohibited Git and filesystem operations

Because the child starts from `implementer/`, Pi loads `implementer/AGENTS.md` but not the sibling `architect/AGENTS.md`.

### 4. Perform the Implementer preflight

Before changing files, the child checks:

```bash
git branch --show-current
git status --short
```

The branch must exactly match the prompt, must not be `main`, and the working tree must be clean. Any mismatch is a blocking failure. The child reports it without editing, staging, cleaning, resetting, restoring, or stashing anything.

### 5. Implement, test, and commit

The Implementer reads `../architect/DESIGN.md`, changes only delegated files in its current workspace, writes or updates tests, and runs the exact test commands. It stages only intentional paths:

```bash
git add src/user.py tests/test_user.py
git commit -m "feat: add validated user model"
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

`RESULT: SUCCESS` means all delegated acceptance criteria are complete and every required test passed. A commit alone does not make work successful or approved.

### 7. Review the checkpoint

After the child exits, control returns to the Architect process in `architect/`. The Architect verifies the exact checkpoint instead of trusting the handoff summary:

```bash
git branch --show-current
git show a1b2c3d
git diff main...implementer/001-user-model
git diff main...implementer/001-user-model -- ':(top)implementer/'
```

The unscoped diff detects any forbidden changes outside the implementation workspace. The scoped diff focuses the code review on `implementer/`.

The Architect confirms that:

- The commit exists and is reachable from the assigned branch.
- All child-authored changes are under `implementer/`.
- `implementer/AGENTS.md` was not changed.
- No root-level or `architect/` file was changed by the child.
- The implementation follows `DESIGN.md` and meets the acceptance criteria.
- Every required test passes when run independently from `../implementer/`.

If review fails, the Architect records the findings and delegates a corrective micro-task on the same branch. It does not merge incomplete or rejected work.

### 8. Update state and merge approved work

After successful review, the Architect updates its local `DESIGN.md` and `TASKS.md` and commits those state changes separately on the implementation branch:

```bash
git add DESIGN.md TASKS.md
git commit -m "docs: record completed user-model task"
git switch main
git merge --no-ff implementer/001-user-model
```

The Architect runs the relevant tests again from `../implementer/` after the merge. It deletes the implementation branch only after the merge and post-merge verification succeed. It never pushes `main` unless the user explicitly requests it.

## Timeout and partial-work recovery

The child has a strict 1,200-second execution window. At approximately 1,000 seconds, it must stop starting new work and enter Wrap-Up mode:

1. Stabilize syntax and leave the implementation in the safest practical state.
2. Run the most relevant required tests that fit within the remaining time.
3. Stage only delegated files under `implementer/`.
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
- Implement refresh-expiry validation.

Working tree:
- Clean.

RESULT: FAILURE
Committed a stable WIP checkpoint; refresh validation remains incomplete.
```

The Architect reviews the checkpoint but does not merge it merely because it is committed. It decomposes the remaining work and starts another child sequentially from `implementer/` on the **same implementation branch**, using the checkpoint as the starting state.

If the child made no repository changes, it should not fabricate an empty commit. It reports `COMMIT: NONE` and explains the blocker.

## Troubleshooting

### The Implementer receives Architect instructions

With this sibling layout, `architect/AGENTS.md` is not an ancestor of `implementer/` and should not be loaded automatically. Confirm that:

- The child was launched from `implementer/`.
- No Architect instructions remain in a root-level `AGENTS.md`.
- No user-level or higher-level Pi context file defines the Architect role.
- The delegated prompt does not assign Architect responsibilities to the child.

### The Architect receives Implementer instructions

Confirm that the parent Pi process was started from `architect/` and that no Implementer contract was copied to the repository root. Pi does not automatically load the sibling `implementer/AGENTS.md`.

### The Architect tells the child not to commit

That instruction violates this workflow. The Implementer owns implementation commits, including stabilized timeout checkpoints. The Architect owns review and integration. Inspect user-level Pi context and the generated task prompt for competing Git rules.

### The child starts on `main` or the wrong branch

The child must stop before editing and return `RESULT: FAILURE`. The Architect should restore a known, clean state without discarding unrelated work, create or check out the intended branch, and delegate again.

### The working tree is unexpectedly dirty

Neither agent should automatically clean, reset, restore, or stash the tree. Determine who owns each change. Commit Architect-owned state separately, preserve unrelated user work, and delegate only from a clean implementation branch.

### Tests pass in the child but fail after merge

The Architect must stop and report the post-merge failure without pushing. Investigate integration differences on `main`; do not conceal the failure or treat the earlier child result as sufficient verification.

## Design trade-offs

This workflow favors traceability, role clarity, and low concurrent resource use over speed:

- Tasks execute sequentially rather than in parallel.
- Each child starts with a fresh conversation and must read relevant project context again.
- Sibling directories prevent project-level role inheritance but do not block deliberate filesystem access; file-ownership rules remain necessary.
- Both agents share one working tree, so a branch switch affects the entire repository. The Architect must wait for the child to finish before switching branches.
- Keeping all implementation tooling under `implementer/` may require path adjustments when adapting an existing project.
- Dedicated branches, commit hashes, and independent review add Git overhead but provide clear provenance and reliable timeout recovery.

For concurrent Implementers or stronger branch isolation, use separate Git worktrees or repository clones. Do not run multiple Implementers concurrently in this shared-working-tree design.

## References

- [Pi repository and documentation](https://github.com/earendil-works/pi)
- [Pi project-context loading](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/resource-loader.ts)
- [Pi Bash timeout implementation](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/bash.ts)

Descriptions of Pi's context loading and timeout behavior are paraphrased from the linked source files.
