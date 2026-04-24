---
name: ralphex-settings
description: View or change ralphex settings (base_branch, codex_model, codex_reasoning_effort)
disable-model-invocation: true
allowed-tools: Bash(*)
---

# Ralphex Settings

Manage the ralphex configuration stored in `.claude/ralphex.local.md`.

## Steps

1. Check if `.claude/ralphex.local.md` exists. If it does, read it and parse the YAML frontmatter to extract the current values of `base_branch`, `codex_model`, and `codex_reasoning_effort`. If it does not exist, use these defaults: `base_branch` = `main`, `codex_model` = `gpt-5.4`, `codex_reasoning_effort` = `high`.

2. Show the user their current settings:

   ```
   **Current ralphex settings:**
   - base_branch: {value}
   - codex_model: {value}
   - codex_reasoning_effort: {value}
   ```

3. Ask the user which settings they want to change using AskUserQuestion.
4. For each selected setting, ask for the new value using AskUserQuestion:
   - For `base_branch`: suggest the current value plus `main`, `master`, `develop` as options.
   - For `codex_model`: suggest `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.5` as options.
   - For `codex_reasoning_effort`: offer `low`, `medium`, `high`, `xhigh` as options with `high` first.
5. Write the updated settings to `.claude/ralphex.local.md` with YAML frontmatter:

   ```markdown
   ---
   base_branch: {value}
   codex_model: {value}
   codex_reasoning_effort: {value}
   ---
   ```

6. Confirm the changes to the user by showing the new settings.
