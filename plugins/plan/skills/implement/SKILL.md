---
name: implement
description: Plan and execute coding work step by step using autonomous agents. Use when the user asks to plan and implement a feature or a new project end-to-end.
---

# Plan and execute coding work step by step

## Workflow

### Step 1: Understand the task
Ask the user what they want to build or implement. Clarify:
- What the feature/change should do
- Constraints (language, framework, compatibility)
- Goals and success criteria

If the user already provided a clear description, confirm your understanding and move on.

### Step 2: Explore context
- If there's an existing codebase, use **Explore agents** to understand relevant code, patterns, architecture, and conventions.
- If it's a greenfield project, research best practices and technologies relevant to the task.
- Summarize what you found for the user.

### Step 3: Design the architecture
Design or adapt the architecture for the task:
- Identify components, data flow, interfaces, directory structure, and technology choices
- For existing codebases, show how the new work fits into what's already there
- For new projects, propose the full architecture

Present the architecture to the user and ask for feedback. Iterate until they're satisfied.

### Step 4: Write a plan
Break the work into discrete, ordered tasks with clear descriptions. Each task should be small enough for a single agent run to complete.

### Step 5: Get user approval
Present the full plan to the user using `AskUserQuestion`. Ask for confirmation or changes. Iterate until approved.

### Step 6: Choose the plan file name
Pick a short kebab-case name for the task (e.g., `add-cli-parser`) and get today's date with `date +%Y-%m-%d`.
Set `PLAN_FILE` to `docs/plans/<date>-<task_name>.md` (e.g., `docs/plans/2026-10-02-add-cli-parser.md`).

### Step 7: Prepare git repository and branch

#### 7a: Ensure git repository exists
Check if the current directory is a git repository (`git rev-parse --is-inside-work-tree`).
If not, initialize one:
1. Run `git init`
2. Analyze the project files and create a `.gitignore` appropriate for the project (languages, frameworks, build artifacts, IDE files, etc.)
3. Stage all files and create an initial commit: `git add -A && git commit -m "Initial commit"`

#### 7b: Check for uncommitted changes
Run `git status` to check for untracked files or uncommitted changes.
- If there are changes that are related to the current plan (e.g., docs/plans file, notes), commit them with an appropriate message and continue.
- If there are unrelated uncommitted changes, print a warning explaining what was found and **stop**. Ask the user to commit or stash their changes before proceeding.
- If the working tree is clean, continue.

#### 7c: Create or reuse a feature branch
Detect the current branch name (`git branch --show-current`).
- If on the default branch (`main` or `master`): create a new feature branch from the plan filename `PLAN_FILE` — use the filename without path, extension and date prefix (e.g., `docs/plans/2026-10-02-add-cli-parser.md` → branch `add-cli-parser`). Run `git checkout -b <branch-name>`.
- If already on a non-default branch: continue on it — assume the user is intentionally working on this branch.

### Step 8: Save the plan

#### 8a: Write the plan file
Create the directory `docs/plans/` if it doesn't exist, then write the approved plan to `PLAN_FILE` using this format:

```markdown
# Feature Name

## Description
What this feature does.

## Context
Language, framework, testing tools, relevant details.

## Tasks

### Task 1: Set up module
- [ ] Create the main file with core functions
- [ ] Add error handling

### Task 2: Add CLI
- [ ] Add argparse interface
- [ ] Print results to stdout

### Task 3: Write tests
- [ ] Test all functions
- [ ] Test error cases
```

#### 8b: Commit the plan
Run `git rev-parse HEAD` and save the result as `START_SHA`. Commit the plan file with the message `Add plan: <task name>`.

### Step 9: Execute via subagents
Run a subagent with the prompt from [coder-prompt.md](coder-prompt.md) with the placeholders filled in.

Check its output:
- `ALL_TASKS_COMPLETED` → move to Step 10
- `TASKS_REMAINING` → repeat Step 9
- `BLOCKED: ...` → stop and ask the user for guidance

If an agent fails 3+ times on the same task, stop and ask the user for guidance.

### Step 10: Review and fix

#### 10a: Review changes
Run a subagent with the prompt from [review-prompt.md](review-prompt.md) with the placeholders filled in.

Check its output:
- `No issues found.` → move to Step 11
- `N issue(s) found.` → go to 10b
- Anything else → stop and ask the user for guidance

#### 10b: Fix issues
Run a subagent with the prompt from [fixer-prompt.md](fixer-prompt.md) with the placeholders filled in. Keep the skipped issues it lists, with their reasons, for the final report.

Check its output:
- `BLOCKED: ...` → stop and ask the user for guidance
- `Another review pass is recommended.` → go to 10a; if this was the 10th pass, stop and ask the user for guidance instead
- `No further review needed.` → move to Step 11
- Anything else → stop and ask the user for guidance

### Step 11: Verify
- Read the plan file to confirm all checkboxes are checked
- Report results to the user: what was built, what files were created/modified, any issues encountered, and every review issue the fixers skipped, with its reason

## Important rules
- Coder and fixer agents run **sequentially**, not in parallel — each builds on prior work
- Each fixer agent gets the review report and the project root
- **Always** get user approval before starting execution (Step 5)
- The plan file is the source of truth for progress — agents read and update it directly
- If something goes wrong, stop and ask the user rather than guessing
- At the start of every reply during the workflow, state the current step, e.g. `**Step 3: Design the architecture**`
