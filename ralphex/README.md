# ralphex

Claude Code plans and implements, Codex reviews. Repeat until clean.

## How it works

1. You describe a coding task
2. Claude Code plans and implements the solution
3. Claude Code commits with a clean workspace
4. Codex CLI reviews the full branch diff
5. If Codex has corrections, Claude Code re-plans and re-implements
6. Loop continues until Codex gives a clean review (LGTM)

## Usage

```
/ralphex Add a REST API endpoint for user registration with validation
```

## Setup

### Prerequisites

- [Codex CLI](https://github.com/openai/codex) installed and configured
- Git repository with a base branch (default: `main`)

### Configuration

Create `.claude/ralphex.local.md` in your project root:

```markdown
---
base_branch: main
codex_model: gpt-5.3-codex
codex_reasoning_effort: high
---
```

| Setting | Description | Default |
|---------|-------------|---------|
| `base_branch` | Branch to diff against for reviews | `main` |
| `codex_model` | Codex model to use for reviews | *(required)* |
| `codex_reasoning_effort` | Reasoning effort for reviews (`low`, `medium`, `high`, `xhigh`) | `high` |

The command will prompt you for these values if the settings file doesn't exist.

## Installation

```bash
claude --plugin-dir /path/to/ralphex
```
