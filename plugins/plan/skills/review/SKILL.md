---
name: review
description: Review the current branch's changes with parallel reviewer agents and report the issues they find, without fixing them. Use when the user asks to review the changes on a branch.
---

# Review the current branch's changes

Run reviewer agents in parallel on the current branch's changes, collect their findings, deduplicate and sort them, and report the result.

This skill only reports problems. Do not edit files, do not run tests, do not commit.

Run every git command and file operation in the current project directory. Call its absolute path `PROJECT_DIR`.

The reviewer definitions live in this skill's own `agents/` directory, one `*-reviewer.md` file per reviewer. Call its absolute path `AGENTS_DIR`.

## Arguments

Optional: the base branch or commit to diff against. If given, use it as `BASE` and skip the detection in step 1.

## Workflow

### 1. Determine the base to diff against

If no base was given, detect the default branch:
```
git remote show origin 2>/dev/null | grep 'HEAD branch' | awk '{print $NF}'
```
If that fails, check which of `main` or `master` exists locally. Call the result `BASE`.

Run `git diff --name-only BASE...HEAD` to check if there are changes. If the output is empty, print "No changes to review.", end with `No issues found.` and stop.

### 2. Discover reviewers

Find all reviewer files matching `AGENTS_DIR/*-reviewer.md`. The reviewer name is the file name without the `-reviewer.md` suffix (e.g. `bug-reviewer.md` → `bug`).

If no files are found, print `No reviewer files found in AGENTS_DIR` (with the actual path) and stop.

### 3. Build reviewer prompts

For each reviewer file, read it and build the prompt in memory as: **the full contents of the file**, followed by the common block below. Replace `{{PROJECT_DIR}}` and `{{BASE}}` in the block with the values you have. Do not write prompts or any other temporary files to disk.

Common block:

```
## How to get the changes
The project is at `{{PROJECT_DIR}}`. Run every command there.
1. Run `git diff {{BASE}}...HEAD` to get the diff of all changes on this branch.
2. Run `git diff --name-only {{BASE}}...HEAD` to get the list of changed files.
3. Read the full content of each changed file to understand the surrounding context.
4. Review the diff within that context, using the focus defined above.

## Reporting rules
- Report only real, confirmed problems you are confident about.
- Do NOT report speculative, hypothetical, or nitpick issues.
- Do NOT fabricate issues to appear thorough.
- For each issue, provide the exact file path, line number, and a concrete fix suggestion.

## Output format
One issue per line:
path/to/file.ext:42:description of the problem and how to fix it

If you found no real issues, output exactly: NO_ISSUES_FOUND

Output ONLY the issues (or NO_ISSUES_FOUND). No preamble, no summary, no markdown formatting.
```

### 4. Launch reviewers in parallel

Launch one general-purpose subagent per reviewer, each with its own composed prompt, running in the foreground so that you wait for its result. Send all launches in a single message so they run in parallel. Do not merge reviewers into one agent.

Wait until every reviewer has returned. Do not proceed to step 5, and do not finish, with reviewers still running.

### 5. Collect and parse results

Each reviewer returns text. Parse each line matching the output format defined in the common block above: `file:line:description`. Ignore lines that are `NO_ISSUES_FOUND` or empty.

If a reviewer's output contains lines that match neither the format nor `NO_ISSUES_FOUND`, record a warning `Reviewer <name> returned unparseable output` and include it in the summary below.

### 6. Deduplicate

Merge issues that describe the same problem in the same place, even if worded differently. Keep the most descriptive version. Keep different problems on the same line as separate issues.

### 7. Sort and report

Sort issues by file path, then by line number. Print a summary (reviewer names are the ones discovered in step 2):

```
## Review Summary

Ran R reviewer(s): <name>, <name>, ...
Found N issue(s) across M file(s):

path/to/file.ext:10: description
path/to/file.ext:25: description
other/file.ext:3: description

Warnings:
Reviewer <name> returned unparseable output:
<raw output>
```

Omit the `Warnings:` section if there are none.

Count N and M after step 6; the header and the final line use the same N. Print nothing between the summary and the final line.

End with exactly one line:
```
N issue(s) found.
```
or, if no issues were found but some reviewers returned unparseable output (K is the number of such reviewers):
```
No issues found, K reviewer(s) returned unparseable output.
```
or, if no issues were found across all reviewers and there are no warnings:
```
No issues found.
```
