# Notes for agents working on this repo

## Writing steps in a skill

A skill's workflow is an algorithm. Every line of a step tells the agent what to do; nothing restates, summarizes or introduces what follows. Reference: `plugins/plan/skills/implement/SKILL.md`.

### Headings
- Every step and sub-step is a command: its title is an imperative phrase (`Design the architecture`, `Prepare the coder prompt`), never a noun (`Architecture design`, `Coder agent prompt`).
- Steps: `### Step N: <Title>`.
- Sub-steps: `#### Na: <Title>` (`#### 7a: Ensure git repository exists`).
- A step with sub-steps starts directly with the first sub-step. No intro line under `### Step N`.

### Body
- Imperative sentences: `Run …`, `Read …`, `Commit …`.
- Give exact commands and file names in backticks (`git branch --show-current`, `docs/plans/<date>-<task_name>.md`).
- Examples go inline in parentheses: `(e.g., \`add-cli-parser\`)`.
- State each fact once. If a limit, condition or transition is defined in one place, do not repeat it elsewhere (e.g., the 5-pass limit lives only in the line that handles the 5th pass).

### Branching on output
List every outcome as `- <exact output> → <action>`, with outputs in backticks, and cover the rest explicitly:
```
- `ALL_TASKS_COMPLETED` → move to Step 10
- `TASKS_REMAINING` → repeat 9b
- `BLOCKED: ...` → stop and ask the user for guidance
- Anything else → stop and ask the user for guidance
```
- Transitions: `move to Step N`; within a step, `go to Na` / `repeat Na`.
- Stops: `stop and ask the user for guidance`.

### Bundled templates
Reference the file as a link and list its placeholders:
```
Read [coder-prompt.md](coder-prompt.md) and fill in the placeholders:
- `{{goal}}` — brief description of the overall goal
- `{{project_dir}}` — absolute path to the project root
```
Placeholders use `{{name}}`; plain variables in step text use bare names (`BASE`, `AGENTS_DIR`).
