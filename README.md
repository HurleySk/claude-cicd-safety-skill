# CI/CD Safety Skills

A [Claude Code](https://claude.ai/claude-code) skill set for protecting production environments from accidental agent modifications.

Born from a real incident where a Claude agent misconfigured environment assignments, causing dev pipelines to target production systems. These skills help you set up deterministic and LLM-powered safety gates.

## Prerequisites

- **Node.js** — Required for JSON parsing in hook scripts. Hooks fail-closed (deny) if node is not available.
- **Bash** — Required for hook execution. On Windows, Git Bash (included with Git for Windows) works.

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

Set up Claude Code PreToolUse hooks that intercept file writes, command execution, and block dangerous changes before they reach production.

| Command | Description |
|---------|-------------|
| `/cicd-safety:safety-hooks setup` | Interactive setup — identifies environments, creates rules, generates hooks |
| `/cicd-safety:safety-hooks audit` | Review existing hooks and report safety gaps |
| `/cicd-safety:safety-hooks status` | Quick view of what's protected |
| `/cicd-safety:safety-hooks test` | Validate that hooks are functioning correctly |
| `/cicd-safety:safety-hooks disable` | Temporarily disable all safety hooks |
| `/cicd-safety:safety-hooks enable` | Re-enable disabled safety hooks |
| `/cicd-safety:safety-hooks uninstall` | Cleanly remove all safety artifacts |
| `/cicd-safety:safety-hooks help` | Show available commands |

**Architecture**: Three-layer defense across all tool types:
1. **Deterministic script** — Fast pattern matching (bash+node). Fail-closed design. Catches known dangerous patterns, secrets/credentials, and environment mismatches.
2. **LLM prompt hook** — Semantic backstop. Catches subtle issues the script misses.
3. **Command confirmation** — Guards Bash and PowerShell: git operations, cloud CLI (terraform, kubectl, az/aws/gcloud), destructive commands (rm -rf), and package installs.

**Tool coverage:** Write, Edit, NotebookEdit, Bash, PowerShell

### `cicd-fortify` — Automated CI/CD Safety Assessment

Dispatches a **Sentinel** agent that deeply analyzes your project's CI/CD posture, using **Scout** agents for fast reconnaissance. Produces a comprehensive assessment with risks, recommendations, and a machine-readable rules file for seamless handoff to `safety-hooks setup`.

| Command | Description |
|---------|-------------|
| `/cicd-safety:cicd-fortify assess` | Full Sentinel/Scout assessment of your project |
| `/cicd-safety:cicd-fortify help` | Show available commands |

**How it works:**
1. **Sentinel** (expert agent) analyzes: environments, deployment paths, safety hooks, risky directories, prod indicators, branch protection
2. **Scouts** (fast haiku agents) gather specific facts in parallel
3. Sentinel produces `output/cicd-fortify-assessment.md` (findings) and `output/cicd-fortify-rules.json` (machine-readable rules)
4. You work through open questions with the orchestrator
5. Implementation via `safety-hooks setup` — pre-populated from the assessment

### `guard` — Pre-Commit Safety Review Agent

A haiku-powered Guard agent that reviews git diffs for safety concerns. Triggered by a PostToolUse hook after commits with risky files. Reads risky patterns from `safety-rules.json` (shared config with safety-hooks).

| Command | Description |
|---------|-------------|
| `/cicd-safety:guard setup` | Set up the Guard reminder hook |
| `/cicd-safety:guard review` | Manually run Guard review on current changes |
| `/cicd-safety:guard help` | Show available commands |

**How it works:** A PostToolUse hook detects commits with risky files (patterns loaded from `safety-rules.json`) and prints a reminder. The agent then invokes `/cicd-safety:guard review`, which dispatches a fast haiku subagent to review the full git diff for environment mismatches, hardcoded prod values, secret exposure, and more.

## Recommended Workflow

1. **Assess** — Run `/cicd-safety:cicd-fortify assess` to understand your CI/CD posture
2. **Fortify** — Use the assessment findings to run `/cicd-safety:safety-hooks setup` (auto-populated)
3. **Guard** — Activate `/cicd-safety:guard setup` for ongoing pre-commit review
4. **Maintain** — Use `status`, `test`, and `audit` to keep hooks healthy over time

## Known Limitations

- **MCP tools** are not intercepted by Claude Code hooks. If you use MCP servers that can modify files or infrastructure, configure safety at the MCP server level.
- **Windows support** requires Git Bash (included with Git for Windows). Hook scripts are bash-based and invoked via the `bash` command.

## License

MIT
