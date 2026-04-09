---
name: guard
description: Use when the user wants to set up a pre-commit Guard agent that reviews staged git changes for safety concerns before committing, or when you want to manually run a Guard review on current changes.
argument-hint: "[setup|review|help]"
---

# Guard — Pre-Commit Safety Review Agent

You are the orchestrator for the Guard system. The Guard is a haiku subagent that reviews staged git changes for production safety concerns before commits happen. It replaces flaky LLM prompt hooks with a deliberate, agent-dispatched review triggered at commit time.

## Why Guard Instead of Prompt Hooks

`type: "prompt"` hooks fire on EVERY file write and often return malformed responses (causing "hook error" banners). The Guard approach is better:
- Runs once at commit time, not on every file write
- Reviews the full changeset (git diff), not individual files in isolation
- The main agent dispatches it deliberately, not as an automatic hook
- Uses the Agent tool which is reliable, unlike prompt hooks which may not output valid JSON

## Commands

### `setup`

Set up the Guard system for this project.

#### Step 1: Create the Guard Hook Script

Write `.claude/hooks/guard-reminder.sh` — a PostToolUse hook on Bash that detects `git commit` and reminds the agent to run the Guard:

```bash
#!/bin/bash
# PostToolUse hook: after a git commit with risky files, remind agent to invoke Guard skill.

INPUT=$(cat)

COMMAND=$(echo "$INPUT" | node -e "
  let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{
    try{console.log(JSON.parse(d).tool_input.command||'')}catch{console.log('')}
  })")

# Only trigger on git commit
if ! echo "$COMMAND" | grep -q 'git commit'; then
  exit 0
fi

# Load risky patterns from safety-rules.json (single source of truth)
RULES_FILE="$CLAUDE_PROJECT_DIR/.claude/safety-rules.json"
if [ -f "$RULES_FILE" ] && command -v node &>/dev/null; then
  # Build grep pattern from riskyDirectories and riskyPatterns in rules file
  PATTERN=$(node -e "
    const r=require('$RULES_FILE');
    const dirs=(r.riskyDirectories||[]).map(d=>d.replace(/\\/$/,'').replace(/[.*+?^$()|[\\]{}]/g,'\\\\$&')+'/');
    const pats=(r.riskyPatterns||[]).map(p=>p.replace(/^\*\\./, '\\\\\\\\.')).filter(Boolean);
    const all=[...dirs,...pats].filter(Boolean);
    console.log(all.join('|')||'__NO_PATTERN__');
  ")
else
  # Fallback: use a default pattern if rules file is unavailable
  # CUSTOMIZE THIS for your project if not using safety-rules.json:
  PATTERN='(deploy/|terraform/|\.sql$|pipeline/)'
fi

RISKY_FILES=$(git diff HEAD~1 --name-only 2>/dev/null | grep -E "$PATTERN" | head -5)

if [ -n "$RISKY_FILES" ]; then
  echo "⚠️ GUARD: Risky files committed. Run /cicd-safety:guard review to check for safety concerns."
fi
```

The hook outputs a short reminder. The agent sees this in the PostToolUse output and should invoke `/cicd-safety:guard review` via the Skill tool. This is more reliable than `type: "prompt"` hooks because the agent dispatches the review deliberately rather than an automatic LLM hook that may malform its response.

#### Step 2: Wire Into settings.json

Read `.claude/settings.json` and add the guard-reminder hook to PostToolUse:

```json
{
  "matcher": "Bash",
  "hooks": [
    {
      "type": "command",
      "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/guard-reminder.sh\""
    }
  ]
}
```

**Important:** Add this alongside existing PostToolUse Bash hooks, don't replace them. If the user already has a post-commit hook, add the guard-reminder as a second entry in the hooks array.

#### Step 3: Customize Risky File Patterns

Ask the user:

> What file patterns indicate risky changes that should trigger a Guard review? (e.g., `*.sql`, `pipeline/*.json`, `deploy/*`, `terraform/*`)

Update the `grep -E` pattern in `guard-reminder.sh` with their patterns.

#### Step 4: Customize Guard Concerns

Ask the user:

> What specific safety concerns should the Guard check for? Beyond the defaults (prod indicators, hardcoded URIs, unverified deployments), are there project-specific patterns?

Update the Guard agent prompt in the reminder script with their concerns.

#### Step 5: Test

1. Make a change to a risky file
2. Commit it
3. Verify the Guard reminder appears in the hook output
4. Dispatch the Guard agent manually to verify it works
5. Make a change to a non-risky file, commit, verify no reminder

### `review`

Manually run a Guard review on current uncommitted or recently committed changes.

#### Step 1: Determine What to Review

Check git status:
```bash
git status --short
git diff --stat
```

If there are staged changes, review those. If the last commit was recent, review `HEAD~1..HEAD`. Ask the user if unclear.

#### Step 2: Dispatch the Guard

Launch an Agent with `model: "haiku"`:

**Guard prompt — fill in `{PROJECT_DIR}`. If `.claude/safety-rules.json` exists, also fill in `{PROD_INDICATORS}` and `{PROD_URI_PATTERNS}` from the rules file to give the Guard project-specific context:**

```
You are the GUARD — a fast, rigorous safety reviewer for CI/CD changes.

Project: {PROJECT_DIR}
Production indicators: {PROD_INDICATORS}
Production URI patterns: {PROD_URI_PATTERNS}

Run `git diff HEAD~1` (or `git diff --cached` for staged changes) and review every changed file.

Check for:
1. **Environment mismatches**: Resources matching the production indicators above assigned to non-prod environments, or dev resources under prod
2. **Hardcoded prod values**: Production URIs, connection strings, or GUIDs that should be parameterized — especially patterns matching the URI patterns above
3. **Secret/credential exposure**: API keys (AKIA...), passwords, tokens, private keys written to tracked files
4. **Unverified deployments**: Deploy steps that aren't preceded by verification steps
5. **Config table dangers**: INSERT/DELETE on config tables without proper WHERE clauses or environment scoping
6. **Protected branch violations**: Changes targeting main/master directly
7. **Dangerous step combinations**: Deploying config changes and immediately executing pipelines without review
8. **Cross-tier resource duplication** (CRITICAL): A resource whose name or URI contains an environment indicator (e.g., "prod", "prd", "dev", "staging", "uat") assigned to a tier that doesn't match — for example, `my-service-prod.example.com` under `environment = "dev"`. This almost always means a copy-paste error.
9. **Shared resources across prod and non-prod** (WARNING): The same URI, connection string, or endpoint appears under both a production tier and a non-production tier. This means a dev/test pipeline run could mutate production data.
10. **Single resource serving many tiers** (INFO): One resource identifier is assigned to 3 or more environment tiers. This may be intentional (shared service) but is worth flagging so the author confirms it.

For each finding, report:
- File and line
- Severity (CRITICAL / WARNING / INFO)
- What's wrong
- Suggested fix

If nothing found, report: "CLEAR — no safety concerns in this changeset."

Be thorough but fast. Only flag real issues, not style preferences.
```

To populate the Guard prompt, read `.claude/safety-rules.json` and extract:
- `prodIndicators` array → join as comma-separated string for `{PROD_INDICATORS}`
- `prodUriPatterns` array → join as comma-separated string for `{PROD_URI_PATTERNS}`

If the rules file doesn't exist, use generic patterns ("prod", "prd", "production") as defaults.

#### Step 3: Report Findings

After the Guard returns:
- If CLEAR: tell the user "Guard review passed — no safety concerns."
- If findings: present each one to the user with the severity and suggested fix. Ask if they want to address any before proceeding.

### `help`

**Available commands:**

| Command | Description |
|---------|-------------|
| `/cicd-safety:guard setup` | Set up the Guard reminder hook for pre-commit safety reviews |
| `/cicd-safety:guard review` | Manually run a Guard review on current changes |
| `/cicd-safety:guard help` | Show this help |

**How it works:**

```
Agent commits code
    │
    ▼
PostToolUse hook fires
    │
    ├── Non-risky files only? → silent (no reminder)
    │
    └── Risky files detected? → prints Guard reminder
                                    │
                                    ▼
                        Agent dispatches Guard (haiku)
                                    │
                                    ▼
                        Guard runs git diff, reviews changes
                                    │
                                    ▼
                        Reports: CLEAR or [findings]
```

**Key design decisions:**
- Guard is a **reminder**, not a blocker. The PostToolUse hook prints a message after the commit; it doesn't prevent it. The agent decides whether to act on it.
- Guard uses **haiku** for speed. A full diff review takes 2-5 seconds, not 30+.
- Guard reviews the **full changeset**, not individual files. This catches cross-file issues that per-file hooks miss.
- The main agent **dispatches** the Guard deliberately via the Agent tool. This is more reliable than `type: "prompt"` or `type: "agent"` hooks which may error.
