---
name: ralphex-review
description: Codex reviews, Claude filters and fixes, repeat until clean
argument-hint: Optionally describe what to focus the review on
allowed-tools: Read, Grep, Glob, Write, Edit, Bash, AskUserQuestion
disable-model-invocation: true
license: MIT
compatibility: Designed for Claude Code. Expects an authenticated Codex CLI installation.
---

# Ralphex Review: Iterative Code Review with Codex

Execute an iterative review loop where Codex reviews your existing code, you (Claude Code) filter the suggestions and implement fixes, then Codex reviews again. Loop until there is no relevant corrections to implement in Codex's review.

---

## Step 1: Read Settings

1. Check if `.claude/ralphex.local.md` exists in the project root.
2. If it exists, read it and parse the YAML frontmatter to extract:
   - `base_branch`: The branch to diff against (default: `main`)
   - `codex_model`: The Codex model to use (REQUIRED - if missing, ask the user)
   - `codex_reasoning_effort`: Reasoning effort for Codex reviews (default: `high`)
3. If the file does not exist, first run `command -v codex` via Bash. If the command exits with a non-zero status (codex not found), inform the user: "Codex CLI is not installed. Install it from https://github.com/openai/codex and try again." and stop. Otherwise, ask the user:
   - What base branch to diff against (suggest `main`)
   - What Codex model to use (`gpt-5.3-codex`, `gpt-5.2-codex`, `gpt-5.3-codex-spark`)
   - What reasoning effort to use for reviews (suggest `high`; valid values: `low`, `medium`, `high`, `xhigh`)
   Then create `.claude/ralphex.local.md` with their answers.\
4. Check for uncommitted changes by running `git status`. If there are uncommitted changes, use `AskUserQuestion` to present these options:
   a. Commit modified tracked files: Run `git add -u` to stage all modified tracked files. Ask the user for a commit message, then commit with that message.
   b. Commit all changes (modified + untracked): Run `git add -A` to stage everything including untracked files. Ask the user for a commit message, then commit with that message.
   c. Stash changes: Generate a unique stash name by running `date +%s` and using `"ralphex-review-stash-{timestamp}"`. Run `git stash push -m "ralphex-review-stash-{timestamp}"`. Store the full stash message for later cleanup.
   d. Cancel: Stop the review.
5. Resolve the review target: Run `git rev-parse --abbrev-ref HEAD` to get the current branch. Compute a `review_target` value:
   - If the current branch is different from `base_branch`, set `review_target` = `base_branch`.
   - If the current branch equals `base_branch`, check if `origin/{base_branch}` exists by running `git rev-parse --verify origin/{base_branch}`. If it exists, set `review_target` = `origin/{base_branch}`. If it does not exist, inform the user that there is no remote to compare against and stop.
   - Verify there is actually a diff by running `git diff --stat {review_target}...HEAD`. If the diff is empty, inform the user there are no changes to review and stop.

Store `review_target`, `codex_model`, and `codex_reasoning_effort` for use throughout the workflow. Use `review_target` (not `base_branch`) in all subsequent git and Codex commands.

---

## Step 2: Codex Code Review

Run Codex to review the branch changes:

1. Run this command via Bash, replacing `{review_target}`, `{codex_model}`, and `{codex_reasoning_effort}` with the resolved values from Step 1, unless the user prompt "$ARGUMENTS" says otherwise:
   ```
   codex exec --sandbox read-only -m {codex_model} -c model_reasoning_effort="{codex_reasoning_effort}" -o .claude/ralphex-review.txt "Review the committed changes of this branch against {review_target}. $ARGUMENTS" 2>/dev/null
   ```

2. Check the exit code. If the command exits with a non-zero status, inform the user that Codex failed (include the exit code). Perform **Cleanup: Restore Stashed Changes** if a stash was created in Step 1, then stop.

---

## Step 3: Evaluate Review

Read the contents of `.claude/ralphex-review.txt`.

### If corrections exist, critically evaluate each one

Do NOT blindly accept all suggestions. Codex can be wrong, pedantic, or suggest changes outside the task scope. For each suggestion in the review, independently evaluate and classify it:

- **Accept**: Real bugs, security issues, correctness problems, or clear improvements directly related to the task. These will be addressed in the next iteration.
- **Reject**: Factually wrong suggestions, misunderstandings of the code or requirements, or purely stylistic preferences that don't improve correctness.
- **Defer**: Valid observations that are out of scope for the current task (e.g., pre-existing issues, unrelated refactoring suggestions).

Display your verdicts to the user in this format:

```
**Iteration {N} - Codex Review Evaluation:**
- ✅ Accept: {summary of accepted suggestion}
- ❌ Reject: {summary and reason for rejection}
- ⏭️ Defer: {summary and reason for deferral}
```

Then decide:

- If **all suggestions are rejected or deferred** → treat this as a clean review. Inform the user. Perform **Cleanup: Restore Stashed Changes** if a stash was created in Step 1, then stop.
- If **any suggestions are accepted** → delete `.claude/ralphex-review.txt` using Bash: `rm .claude/ralphex-review.txt`, then proceed to **Step 4: Implement** with only the accepted corrections. Do not address rejected or deferred items.

### If clean

- Inform the user that Codex approved the implementation
- Display the total number of iterations it took
- Delete `.claude/ralphex-review.txt` using Bash: `rm .claude/ralphex-review.txt`
- Perform **Cleanup: Restore Stashed Changes** if a stash was created in Step 1.

---

## Step 4: Implement

Implement the accepted corrections:

1. For each accepted correction, explore the relevant code to understand the context.
2. Make the necessary code changes. Keep changes focused: only address accepted corrections, do not refactor unrelated code.
3. Verify changes are consistent with the rest of the codebase.

---

## Step 5: Commit

Create a clean git state for Codex to review again:

1. Stage all changed files with `git add` (be specific about which files. Do NOT use `git add .` or `git add -A`).
2. Commit with a descriptive message summarizing what was fixed based on the Codex review.
3. Verify changes are committed with `git status`.

After committing, go back to **Step 2: Codex Code Review** for the next iteration.

---

## Cleanup: Restore Stashed Changes

If a stash was created in Step 1, restore it before stopping:

1. Run `git stash list` and find the entry whose message contains the stash name stored in Step 1 (e.g., `"ralphex-review-stash-{timestamp}"`).
2. Extract its `stash@{N}` reference from the matching line.
3. Run `git stash pop stash@{N}`.
4. If the pop fails due to conflicts, inform the user that the stash could not be applied cleanly and tell them to resolve conflicts and run `git stash pop` manually.

---

## Important Rules

- **Never skip the commit step.** Codex reviews committed diffs.
- **Track iteration count.** Display it with each review cycle so the user knows progress.
- **The loop is fully automatic.** No user approval is needed between iterations. Only stop when Codex gives a clean review, or all suggestions are rejected/deferred.
- **If Codex CLI fails** (command not found, network error, etc.), inform the user and stop. Do not retry automatically.
- **The workspace must be clean before the first review.** Codex reviews committed diffs. If there are uncommitted changes, offer to commit them, stash them, or cancel.
- **If changes were stashed, always restore them.** Every exit path must perform the **Cleanup: Restore Stashed Changes** step before stopping.
- **Clean up** `.claude/ralphex-review.txt` after reading it in every iteration, whether the review is clean or not.
