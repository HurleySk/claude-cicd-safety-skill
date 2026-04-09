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

### `cicd-fortify` — Automated CI/CD Safety Assessment

Dispatches a **Sentinel** agent that deeply analyzes your project's CI/CD posture, using **Scout** agents for fast reconnaissance. Produces a comprehensive assessment with risks, recommendations, and open questions.

| Command | Description |
|---------|-------------|
| `/cicd-safety:cicd-fortify assess` | Full Sentinel/Scout assessment of your project |
| `/cicd-safety:cicd-fortify help` | Show available commands |

**How it works:**
1. **Sentinel** (expert agent) analyzes: environments, deployment paths, safety hooks, risky directories, prod indicators, branch protection
2. **Scouts** (fast haiku agents) gather specific facts in parallel
3. Sentinel produces `output/cicd-fortify-assessment.md` with findings and plan
4. You work through open questions with the orchestrator
5. Implementation via `safety-hooks setup` skill

### `guard` — Pre-Commit Safety Review Agent

A haiku-powered Guard agent that reviews git diffs for safety concerns. Triggered by a PostToolUse hook after commits with risky files.

| Command | Description |
|---------|-------------|
| `/cicd-safety:guard setup` | Set up the Guard reminder hook |
| `/cicd-safety:guard review` | Manually run Guard review on current changes |
| `/cicd-safety:guard help` | Show available commands |

**How it works:** A PostToolUse hook detects commits with risky files and prints a reminder. The agent then invokes `/cicd-safety:guard review`, which dispatches a fast haiku subagent to review the full git diff. Replaces unreliable LLM prompt hooks with a deliberate, agent-dispatched review.

## License

MIT
