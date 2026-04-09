# CI/CD Safety Skills

A [Claude Code](https://claude.ai/claude-code) skill set for protecting production environments from accidental agent modifications.

Born from a real incident where a Claude agent misconfigured environment assignments, causing dev pipelines to target production systems. These skills help you set up deterministic and LLM-powered safety gates.

## Installation

### Via Marketplace

```bash
claude plugin marketplace add HurleySk/claude-plugins-marketplace
claude plugin install cicd-safety
```

### Direct

```bash
claude plugin add HurleySk/claude-cicd-safety-skill
```

## Skills

### `safety-hooks` — Production Safety Hook Setup

Set up Claude Code PreToolUse hooks that intercept file writes and block dangerous changes before they reach production.

| Command | Description |
|---------|-------------|
| `/cicd-safety:safety-hooks setup` | Interactive setup — identifies environments, creates rules, generates hooks |
| `/cicd-safety:safety-hooks audit` | Review existing hooks and report safety gaps |
| `/cicd-safety:safety-hooks help` | Show available commands |

**Architecture**: Three-layer defense:
1. **Deterministic script** — Fast pattern matching (bash+node). Catches known dangerous patterns instantly.
2. **LLM prompt hook** — Semantic backstop. Catches subtle issues the script misses.
3. **Bash confirmation** — Confirms destructive git operations.

## License

MIT
