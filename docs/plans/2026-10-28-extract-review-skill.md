# Extract review skill

## Description
Extract the code review out of the `implement` skill into a standalone `review` skill in the same `plan` plugin (`/plan:review`).

- `review` only finds problems: it runs the reviewer agents on the branch diff, deduplicates and sorts their findings and returns a report. It never edits files, runs tests or commits.
- `implement` calls `plan:review` via the Skill tool in its review step; if issues were found, it spawns a fixer agent that fixes them, runs tests, commits and says whether another review pass is needed.
- The current `implement/agents/review-orchestrator.md` is split: steps 1–7 go into `review/SKILL.md`, steps 8–11 go into `implement/fixer-prompt.md`; the orchestrator file is deleted.

## Context
- Repository: Claude Code plugin marketplace `pragmata-ai`, plugin at `plugins/plan/`, skills at `plugins/plan/skills/<name>/SKILL.md`.
- Content is Markdown and JSON only; there is no test suite. Instead of tests, verify with `claude plugin validate .` from the repo root and check that every file referenced from a skill exists.
- Keep the style of the existing files: English, short imperative instructions, `{{placeholder}}` syntax as in `coder-prompt.md`.
- Use `git mv` for moved files so history is kept.

## Tasks

### Task 1: Create the review skill
- [x] `git mv` the five `plugins/plan/skills/implement/agents/*-reviewer.md` files to `plugins/plan/skills/review/agents/` without changing their content
- [x] Create `plugins/plan/skills/review/SKILL.md` with frontmatter (`name: review`, a `description` saying it reviews the current branch's changes with parallel reviewer agents and reports issues without fixing them). Body: steps 1–7 of `implement/agents/review-orchestrator.md`, adapted to run in the main context (no orchestrator subagent):
  - optional argument: base branch or commit to diff against; if not given, detect the default branch as the orchestrator does now
  - reviewers are discovered in this skill's own `agents/` directory; pass its absolute path where needed
  - keep the common prompt block, the parallel foreground launch, parsing, deduplication and sorting exactly as they are now
  - output the `## Review Summary` in the current format and end with exactly one line: `N issue(s) found.` or `No issues found.`
  - state explicitly: do not edit files, do not run tests, do not commit

### Task 2: Add the fixer prompt to implement
- [x] Create `plugins/plan/skills/implement/fixer-prompt.md` with exactly this content (agreed with the user):

````markdown
# Fixer Agent Prompt Template

You are a fixer agent. Your job is to fix the issues found by the code review.

## Context
- Project root: `{{project_dir}}` — run every command and file operation there.
- Review report:

```
{{issues}}
```

## Instructions

1. Fix all reported issues. For each issue, read the file, apply the fix, and move on.
   Use good judgment — if a suggested fix would break other code, skip it and note why.
2. Run the project's test suite. Detect the test runner:
   - If `pytest.ini`, `pyproject.toml` with pytest config, or `tests/` directory exists → `python -m pytest`
   - If `package.json` with test script exists → `npm test`
   - If `Cargo.toml` exists → `cargo test`
   - If `go.mod` exists → `go test ./...`
   - Otherwise, skip tests and note that no test runner was detected.

   If tests fail, analyze the failures, fix the code, and rerun. Keep iterating until **all tests pass**.
3. Commit the fixes with a short descriptive message.
4. Print `Fixed N issue(s), all tests passing.` and list skipped issues with reasons, if any.

Then state whether another review pass is needed. The reviewers saw the code before your fixes; nobody has reviewed the fixes themselves.

Another pass is recommended if any of:
- a fix changed logic: conditions, control flow, algorithms, error handling
- a fix changed a public interface (signatures, exported names, output format) and spread across several files
- a fix touched files that were not in the original diff
- tests failed after the fixes and needed further fixing
- the review report has warnings about unparseable reviewer output (its findings may be lost)

No further pass is needed if:
- all fixes were cosmetic: comments, docs, local names, formatting, dead code removal
- every fix was local, a few lines, and tests passed on the first run

End with exactly one of:
```
Another review pass is recommended.
No further review needed.
```

## Rules
- Fix only the reported issues; do not modify unrelated code
- If you encounter a blocker you cannot resolve, output: `BLOCKED: <description of the issue>`
````

### Task 3: Rewrite the review step in implement
- [x] Rewrite Step 10 of `plugins/plan/skills/implement/SKILL.md` as a loop of up to 5 passes: invoke `plan:review` via the Skill tool; on `No issues found.` go to Step 11; otherwise read `fixer-prompt.md`, fill `{{issues}}` with the full review report and `{{project_dir}}`, and spawn a `general-purpose` subagent with it. On `BLOCKED: ...` stop and ask the user; on `Another review pass is recommended.` run the next pass; on `No further review needed.` go to Step 11. If the 5th pass still recommends another one, stop and ask the user
- [x] Delete `plugins/plan/skills/implement/agents/review-orchestrator.md` and the then-empty `implement/agents/` directory; remove all remaining references to the orchestrator and `{{AGENTS_DIR}}` from `implement/SKILL.md`
- [x] Add to "Important rules" in `implement/SKILL.md`: at the start of every reply during the workflow, state the current step, e.g. `**Step 3: Architecture design**`

### Task 4: Version and descriptions
- [ ] `plugins/plan/.claude-plugin/plugin.json`: set `version` to `0.0.2`; update `description` to mention the review skill
- [ ] `.claude-plugin/marketplace.json`: update the plan plugin's `description` the same way
- [ ] Run `claude plugin validate .` from the repo root and make sure it passes
