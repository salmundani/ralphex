---
description: View or change ralphex settings (base_branch, codex_model, codex_reasoning_effort)
allowed-tools: Read, Write, AskUserQuestion
---

# Ralphex Settings

Manage the ralphex configuration stored in `.claude/ralphex.local.md`.

## Steps

1. Check if `.claude/ralphex.local.md` exists. If it does, read it and parse the YAML frontmatter to extract the current values of `base_branch`, `codex_model`, and `codex_reasoning_effort`. If it does not exist, use these defaults: `base_branch` = `main`, `codex_model` = empty, `codex_reasoning_effort` = `high`.

2. Show the user their current settings:

```
**Current ralphex settings:**
- base_branch: {value}
- codex_model: {value}
- codex_reasoning_effort: {value}
```

3. Ask the user which settings they want to change using AskUserQuestion. Offer these options:
   - **base_branch** — currently: `{value}`
   - **codex_model** — currently: `{value}`
   - **codex_reasoning_effort** — currently: `{value}`
   - **All looks good** — no changes needed

   Use `multiSelect: true` so they can change multiple settings at once.

4. If the user selected "All looks good", inform them no changes were made and stop.

5. For each selected setting, ask for the new value using AskUserQuestion:
   - For `base_branch`: suggest the current value plus `main`, `master`, `develop` as options.
   - For `codex_model`: suggest `gpt-5.3-codex`, `gpt-5.2-codex`, `gpt-5.3-codex-spark` as options.
   - For `codex_reasoning_effort`: offer `low`, `medium`, `high`, `xhigh` as options with `high` first.

6. **Validate** each new value:
   - `base_branch` and `codex_model` must match `^[a-zA-Z0-9._/-]+$`. If invalid, reject and re-ask.
   - `codex_reasoning_effort` must be one of: `low`, `medium`, `high`, `xhigh`.

7. Write the updated settings to `.claude/ralphex.local.md` with YAML frontmatter:

```markdown
---
base_branch: {value}
codex_model: {value}
codex_reasoning_effort: {value}
---
```

8. Confirm the changes to the user by showing the new settings.
