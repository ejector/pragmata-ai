---
name: review
description: Review the changes on a branch with a reviewer agent and report the issues it finds, without fixing them, when the user asks for a review.
---

# Review the current branch's changes

This skill only reports problems. Do not edit files, do not run tests, do not commit.

Run every git command and file operation in the current project directory. Call its absolute path `PROJECT_DIR`.

## Arguments

Optional: the base branch or commit to diff against. If given, use it as `BASE` and skip the detection in Step 1.

## Workflow

### Step 1: Determine the base to diff against

If no base was given, detect the default branch:
```
git remote show origin 2>/dev/null | grep 'HEAD branch' | awk '{print $NF}'
```
If that fails, check which of `main` or `master` exists locally. Call the result `BASE`.

Run `git diff --name-only BASE...HEAD` to check if there are changes. If the output is empty, print `No changes to review.`, end with `No issues found.` and stop.

### Step 2: Run the reviewer

Run a subagent on the `opus` model with the prompt from [reviewer-prompt.md](reviewer-prompt.md) with the placeholders filled in.

### Step 3: Report

Print the reviewer's blocks unchanged, sorted by severity (high first), then by file path and line. N is the number of blocks, M the number of files they name:

```
## Review Summary

Found N issue(s) across M file(s):

Location: path/to/file.ext:10
Severity: high
Problem: …
Evidence: …
Fix: …
```

End with exactly one line:
- Issues found → `N issue(s) found.`
- No issues → `No issues found.`
