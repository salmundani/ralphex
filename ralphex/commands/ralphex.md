---
description: Iterative dev loop - Claude Code implements, Codex reviews, repeat until clean
argument-hint: Describe the coding task to implement
allowed-tools: ["Read", "Write", "Edit", "Grep", "Glob", "Bash", "AskUserQuestion"]
---

# Ralphex: Iterative Development with Codex Code Review

Execute an iterative development loop where you (Claude Code) plan and implement code, then Codex reviews your work. Loop until Codex gives a clean review.

**User's task:** $ARGUMENTS

---

## Step 0: Read Settings

**Prerequisite check**: Before doing anything else, verify that Codex CLI is installed by running `command -v codex` via Bash. If the command exits with a non-zero status (codex not found), inform the user: "Codex CLI is not installed. Install it from https://github.com/openai/codex and try again." Then stop immediately.

1. Check if `.claude/ralphex.local.md` exists in the project root.
2. If it exists, read it and parse the YAML frontmatter to extract:
   - `base_branch`: The branch to diff against (default: `main`)
   - `codex_model`: The Codex model to use (REQUIRED - if missing, ask the user)
   - `codex_reasoning_effort`: Reasoning effort for Codex reviews (default: `high`)
3. If the file does not exist, ask the user:
   - What base branch to diff against (suggest `main`)
   - What Codex model to use (`gpt-5.3-codex`, `gpt-5.2-codex`, `gpt-5.3-codex-spark`)
   - What reasoning effort to use for reviews (suggest `high`; valid values: `low`, `medium`, `high`, `xhigh`)
   Then create `.claude/ralphex.local.md` with their answers.

4. **Validate settings values**:
   - `base_branch` and `codex_model` must contain only alphanumeric characters, hyphens, underscores, dots, and forward slashes (regex: `^[a-zA-Z0-9._/-]+$`). If either value contains other characters, reject it and ask the user for a valid value. This prevents shell injection.
   - **Check for reasoning effort in model name**: If `codex_model` ends with `-low`, `-medium`, `-high`, or `-xhigh`, strip the suffix from `codex_model`, use it as `codex_reasoning_effort` (unless already set separately), and update the settings file.
   - `codex_reasoning_effort` must be one of: `low`, `medium`, `high`, `xhigh`. If it contains any other value, reject it and ask the user for a valid value. Default to `high` when not set.

5. **Resolve the review target**: Run `git rev-parse --abbrev-ref HEAD` to get the current branch. Compute a `review_target` value:
   - If the current branch is different from `base_branch`, set `review_target` = `base_branch`.
   - If the current branch equals `base_branch`, check if `origin/{base_branch}` exists by running `git rev-parse --verify origin/{base_branch}`. If it exists, set `review_target` = `origin/{base_branch}`. If it does not exist, inform the user that there is no remote to compare against and stop.
   - Verify there is actually a diff by running `git diff --stat {review_target}...HEAD`. If the diff is empty, inform the user there are no changes to review and stop.

Store `review_target`, `codex_model`, and `codex_reasoning_effort` for use throughout the workflow. Use `review_target` (not `base_branch`) in all subsequent git and Codex commands.

---

## Step 1: Plan

This step has two modes depending on whether this is the first iteration or a subsequent one.

### First iteration — Structured Planning

On the first iteration, go through all four phases before proceeding to implementation.

**Phase 1: Initial Understanding**

Explore the codebase to understand the user's request in context:

1. Use Glob to find relevant files and understand the project structure.
2. Use Grep to search for existing patterns, utilities, and functions related to the task.
3. Use Read to examine the most relevant files in detail.
4. Build a mental model of how the existing code works and where the task fits in.
5. If the user's requirements are unclear or ambiguous, ask clarifying questions via AskUserQuestion before proceeding.

**Phase 2: Design**

Based on Phase 1 findings, design a concrete implementation approach:

1. Identify which files need to be created or modified.
2. Determine what existing patterns, utilities, and conventions to follow.
3. Note any existing code that can be reused.
4. Keep the design focused on the minimum changes needed to accomplish the task.

**Phase 3: Review**

Review your own design for soundness:

1. Check that the approach aligns with the user's task description.
2. Verify you haven't overlooked relevant existing code or patterns.
3. If you have remaining uncertainties or see multiple valid approaches, ask the user via AskUserQuestion.

**Phase 4: Final Plan**

Present the implementation plan to the user for approval:

1. Summarize the recommended approach.
2. List the files to be modified or created.
3. Mention existing utilities or patterns you will reuse.
4. Describe how the changes can be verified.
5. **Ask the user for approval before proceeding to implementation.** Use AskUserQuestion with options like "Approve plan", "Request changes", etc.
6. If the user requests changes, revise the plan and re-present it. Do not proceed until the user approves.

### Subsequent iterations — Lightweight Re-plan

On subsequent iterations (after a Codex review with accepted corrections):

1. Incorporate only the accepted Codex corrections into a targeted re-plan.
2. Avoid full re-exploration, but do focused exploration (Read, Grep, Glob) when accepted corrections reference files or code paths not inspected in the first iteration.
3. No user approval is needed — proceed immediately to implementation.
4. Present the re-plan briefly to the user (just show what you will fix and proceed).

---

## Step 2: Implement

Execute the implementation plan:

1. Write or modify the necessary code files.
2. Make sure the implementation is complete and functional.
3. If you encounter issues, resolve them as part of the implementation.

---

## Step 3: Commit

Create a clean git state for Codex to review:

1. Stage all changed files with `git add` (be specific about which files - do NOT use `git add .` or `git add -A`). **Never stage `.claude/` directory** - it contains local settings that should not be committed.
2. Commit with a descriptive message. Use this format:
   - First iteration: A message describing the initial implementation
   - Subsequent iterations: A message describing what was fixed based on Codex review
3. Verify the working tree is clean with `git status`. There should be no uncommitted changes.

**IMPORTANT**: The workspace MUST be completely clean before proceeding to the review step. Codex reviews the committed diff, so any uncommitted changes will be missed.

---

## Step 4: Codex Code Review

Run Codex to review the branch changes:

1. Run this exact command via Bash:

```
codex exec --sandbox read-only -m {codex_model} -c model_reasoning_effort="{codex_reasoning_effort}" -o .claude/ralphex-review.txt "Review the changes of this branch against {review_target}. If you find issues, bugs, improvements, or corrections, describe each one clearly. If the code looks good and you have no corrections, respond ONLY with the exact text: LGTM" 2>/dev/null
```

Replace `{review_target}`, `{codex_model}`, and `{codex_reasoning_effort}` with the resolved values from Step 0.

2. **Check the exit code.** If the command exits with a non-zero status, inform the user that Codex failed (include the exit code) and stop. Do not attempt to read the review file.

3. Read the file `.claude/ralphex-review.txt` using the Read tool.

---

## Step 5: Evaluate Review

Determine if the review is clean or has corrections:

1. Read the contents of `.claude/ralphex-review.txt`.
2. The review is **an error** if:
   - The file is empty or does not exist (this means Codex failed silently - treat as error, inform the user, and stop)
3. The review is **clean** (no corrections needed) if:
   - The file contains only "LGTM" (or very close variations)
4. The review **has corrections** if:
   - It contains specific issues, bugs, suggestions, or improvements
   - It describes code changes that should be made

### If corrections exist — critically evaluate each one:

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
- If **all suggestions are rejected or deferred** → treat this as a clean review. Inform the user and stop.
- If **any suggestions are accepted** → delete `.claude/ralphex-review.txt` using Bash: `rm .claude/ralphex-review.txt`, then go back to **Step 1: Plan** with only the accepted corrections as context. Do not address rejected or deferred items.

### If clean (LGTM):
- Inform the user that Codex approved the implementation
- Display the total number of iterations it took
- Delete `.claude/ralphex-review.txt` using Bash: `rm .claude/ralphex-review.txt`
- You are done!

---

## Important Rules

- **Never skip the commit step.** Codex reviews the committed diff, so the workspace must be clean.
- **Let Codex review against the base branch.** It will determine the diff itself.
- **Track iteration count.** Display it with each review cycle so the user knows progress.
- **The initial plan requires user approval.** Do not begin implementation until the user approves the plan from Step 1, Phase 4.
- **Do NOT ask for user approval** between subsequent iterations. After the first implementation cycle, the loop is automatic. Only stop when Codex gives LGTM.
- **If Codex CLI fails** (command not found, network error, etc.), inform the user and stop. Do not retry automatically.
- **Clean up** `.claude/ralphex-review.txt` after reading it in every iteration, whether the review is clean or not.
