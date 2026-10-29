# Notes for agents working on this repo

## Writing steps in a skill

A skill's workflow is an algorithm. Every line of a step tells the agent what to do; nothing restates, summarizes or introduces what follows. Reference: `plugins/plan/skills/implement/SKILL.md`.

### Headings
- Every step and sub-step is a command: its title is an imperative phrase (`Run the tests`), never a noun (`Test run`).
- Steps: `### Step N: <Title>`.
- Sub-steps: `#### Na: <Title>` (`#### 2a: Install dependencies`).
- A step with sub-steps starts directly with the first sub-step. No intro line under `### Step N`.

### Body
- Imperative sentences: `Run …`, `Read …`, `Commit …`.
- Give exact commands and file names in backticks (`git status`, `<dir>/<name>.md`).
- Examples go inline in parentheses: `(e.g., \`my-feature\`)`.
- State each fact once. If a limit, condition or transition is defined in one place, do not repeat it elsewhere (e.g., a pass limit lives only in the line that handles the last pass).

### Branching on output
An agent whose output is branched on ends with exactly one marker line, with nothing after it. The step checks the last line of the output.

List every outcome as `- <exact output> → <action>`, with outputs in backticks, and cover the rest explicitly:
```
- `DONE` → move to Step N
- `RETRY` → repeat Step N
- `BLOCKED: ...` → stop and ask the user for guidance
- Anything else → stop and ask the user for guidance
```
- Transitions: `move to Step N` / `repeat Step N`; within a step, `go to Na` / `repeat Na`.
- Stops: `stop and ask the user for guidance`.

### Bundled templates
Reference the file as a link and let the agent fill the placeholders from context:
```
Run a subagent with the prompt from [<name>-prompt.md](<name>-prompt.md) with the placeholders filled in.
```
Placeholders use `{{name}}`; plain variables in step text use bare names (`OUTPUT_DIR`).

## Writing agent prompts
- Ask agents to check state by re-reading files, not from memory (`Re-read <file> and check …`).
- Tell agents where to put explanations, or they append them to the marker line.
- To call a skill from another skill's workflow, run it in a subagent whose prompt tells it to invoke the skill and output the skill's result unchanged. A skill invoked inline puts its text into the caller's context, and its closing line can end the caller's run.

## Developing plugins
- Layout: the marketplace is `.claude-plugin/marketplace.json`; each plugin lives in `plugins/<plugin>/` with `.claude-plugin/plugin.json` and `skills/<skill>/SKILL.md`. Files a skill uses sit next to its `SKILL.md`.
- Keep a plugin's `description` the same in `plugin.json` and `marketplace.json`.
- Installed plugins are cached by version. Bump `version` in `plugin.json` for every change that should reach users.
- After a merge, update the installed copy: `claude plugin marketplace update <marketplace>`, then `claude plugin update <plugin>@<marketplace>`.

## Testing a skill
- Do not test on the installed plugin. Copy the plugin to the scratchpad, rename it in its `plugin.json` (e.g., `<plugin>-dev`), and replace `<plugin>:` with `<plugin>-dev:` in its files.
- Run the skill on a small scratch project, on Sonnet, so that ambiguous wording shows up:
  ```
  claude -p "<prompt>" --model sonnet --plugin-dir <copy> --allowedTools <tools> --output-format stream-json --verbose > run.jsonl
  ```
  Never use `--dangerously-skip-permissions`.
- Read `run.jsonl`: events with `parent_tool_use_id` come from subagents; `Agent` tool calls hold the filled-in prompts; the `result` event holds `total_cost_usd`.
- Check that every marker line is exact, no `{{…}}` is left in prompts, and the run reaches the last step.
