# Sequential Pi Multi-Agent Architecture Setup

This guide provides the step-by-step instructions to set up a parent-child multi-agent system using Pi, optimized for local hardware constraints (like Qwen 3.8 q8 with a 64k context window).
This system enforce a sequential execution between the two agents, so that the local model is never under pressure.
The main advantage is that the parent "architect" agent keeps a lightweight overall view of the macro-task without bloating the context window with implementation details. On the other side, the child "implementer" agent get enough context to the carry out the narrow and focused implementation task that has been handed off to it, without bloating its context window with information that are not strictly related to the task.

## Step 1: Create the Project Structure

First, set up your root directory and the source code sub-folder.
This isolation is critical for separating the parent and child agent instructions.

```bash
mkdir my-pi-project
cd my-pi-project
mkdir src
touch TASKS.md AGENTS.md src/AGENTS.md
```

Your directory tree should look like this:

```text
[root]
├── AGENTS.md        (Parent Agent Config)
├── TASKS.md         (State & Prompt Tracker)
└── src/
    └── AGENTS.md    (Child Agent Config)
```

## Step 2: Install the Timeout Extension

Install the extension that enables the bash timeout configuration.
Run this command from your terminal:

```bash
pi install npm:@piotr-oles/pi-bash-timeout
```

## Step 3: Configure the Timeout via a Shell Function

The `@piotr-oles/pi-bash-timeout` extension resolves its configuration from the
environment (or CLI flags) at startup, before any project settings file would be
loaded. Because of this, the timeout values must already be present in the
environment when you launch `pi`. Its built-in defaults are a 120s default and a
600s maximum, so any requested timeout above 600s gets capped unless you raise
the maximum.

To make `pi` always launch with the raised timeouts, define a `pi` shell function
in your `~/.zshrc` that injects the environment variables:

```zsh
# in ~/.zshrc
pi() {
  PI_BASH_DEFAULT_TIMEOUT_SECONDS=1200 \
  PI_BASH_MAX_TIMEOUT_SECONDS=1300 \
  command pi "$@"
}
```

Notes:

- `command pi` calls the real `pi` binary and avoids the function calling itself.
- `"$@"` forwards any flags you pass (e.g. `pi -p "..."`).
- Environment variables are inherited by child processes, so a child agent spawned
  from `src/` picks up the same timeouts as the parent.
- The maximum is set slightly above 1200 so parent/child startup and shutdown
  overhead does not race against the outer timeout.

Reload your shell after editing:

```zsh
source ~/.zshrc
```

## Step 4: Configure the Parent Agent (The Architect)

Open **[root]/AGENTS.md** and insert the following system instructions.
This configures the parent to act as a planner, state manager, and delegator.

```text
You are the Lead Architect Agent. Your role is to analyze the codebase, define the testing strategy, decompose tasks into `TASKS.md`, and delegate execution.
Rules:
- **Filtered Analysis:** Always begin by scanning the project context using: `find src -type f -not -name "AGENTS.md" -exec cat {} +`
- **Test Discovery & Decision:** Based on your analysis, determine the existing testing framework. If none exists, select the most appropriate standard framework for the language. 
- **Task Management:** Draft all micro-tasks in `TASKS.md`. You must declare the chosen testing framework at the top of the file. Format each micro-task as a checklist item with a specific prompt.
- **Test-Driven Prompts:** Every task prompt assigned to the child MUST include explicit instructions on what tests to write and the exact terminal command to run them.
- **Delegation:** Spawn the child agent using `cd src && pi -p "[PROMPT FROM TASKS.md]"`. Explicitly set the bash tool call's timeout parameter to 1200 to accommodate local hardware execution. Wait for the call to finish.
- **State Updates:** 
  - On "RESULT: SUCCESS", mark the task `[x]` in `TASKS.md`.
  - On "RESULT: FAILURE", analyze the child's error summary and rewrite the remaining uncompleted tasks/prompts in `TASKS.md` with a new technical or testing approach before delegating again.
- Do NOT write or edit production code yourself.
```

## Step 5: Configure the Child Agent (The Implementer)

Open **[root]/src/AGENTS.md** and insert the following instructions. This restricts the child to executing specific prompts, running tests, and outputting a clean exit state for the parent.

```text
You are a Code Implementer. Make specific code changes in this directory based on the prompt, write the requested tests, and verify your results.
Rules:
- Focus exclusively on the code changes and tests required by the parent's prompt.
- Do not modify `AGENTS.md` under any circumstances.
- **Mandatory Testing:** You must execute the test commands provided in your prompt. 
- **Exit State:** At the very end of your execution, you MUST output an exit state. 
  - Output exactly "RESULT: SUCCESS" only if all required tests pass.
  - Output exactly "RESULT: FAILURE" if the tests fail or you encounter an unresolvable error. 
  - Follow the exit state with a 1-sentence summary of the outcome or the specific test failure.
- Avoid large, verbose context transfers; keep your final output as concise as possible.
```

## Step 6: Initialize the Task Tracker

Open **[root]/TASKS.md**. You can leave this blank for the parent to fill out entirely, or you can seed it with your macro-task goal at the top to give the parent immediate context upon startup.

```text
# Macro-Task: [Insert your overall goal here, e.g., Implement a Python User Auth API]
**Testing Framework:** [To be determined by Architect]

- [ ] Task 1: 
  - Prompt: 
```

## Step 7: Launch the System

From your [root] directory, simply start the Pi agent:

```bash
pi
```

Once the interface loads, ask the agent to "Begin working on the macro-task defined in TASKS.md."
The parent will scan the directory, populate the task list, and sequentially spawn child agents to handle the implementation!
