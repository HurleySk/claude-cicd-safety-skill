---
name: safety-hooks
description: Use when the user wants to set up production safety hooks, add deployment guardrails, protect environments from accidental agent modifications, or audit existing safety configurations in Claude Code projects.
argument-hint: "[setup|audit|help]"
---

# Production Safety Hooks

You are an expert at setting up Claude Code hooks that prevent agents from accidentally modifying production environments. Follow the workflows below to create a layered safety system: deterministic script checks + LLM semantic review.

## Background

Claude Code hooks intercept tool calls before or after execution. A `PreToolUse` hook on `Write|Edit` can inspect file content before it's written and `deny` (hard block), `ask` (user confirms), or `allow` (silent pass) the operation.

**Three hook types available:**
- `type: "command"` — Runs a shell script. Returns JSON with `permissionDecision`. Fast and deterministic.
- `type: "prompt"` — Sends content to an LLM for evaluation. Returns `{"ok": true}` or `{"ok": false, "reason": "..."}`. Good for semantic analysis.
- `type: "agent"` — Spawns a full agent with tool access. Powerful but may error in some environments; prefer `prompt` for reliability.

**Hook response formats (critical — using the wrong format causes "hook error"):**

| Hook type | Allow | Block | Confirm |
|-----------|-------|-------|---------|
| `command` | `exit 0` (no output) | `{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny","permissionDecisionReason":"..."}}` | Same but `"permissionDecision":"ask"` |
| `prompt` | `{"ok": true}` | `{"ok": false, "reason": "..."}` | N/A (only ok/not-ok) |

## Commands

### `setup`

Interactive safety hook setup. Walk through each step with the user.

#### Step 1: Identify Environments

Ask the user:

> What environments does your project have? For each, what names identify them in your config/code?
>
> Example: `dev` (factory: dev1, connection: dev-sql), `staging` (factory: qa, connection: qa-sql), `prod` (factory: prd, connection: prod-sql)

Record:
- `PROD_NAMES` — environment names, factory names, connection names that indicate production
- `DEV_NAMES` — names that indicate development/test
- `PROD_URI_PATTERNS` — URL patterns, hostnames, org IDs that identify production resources

#### Step 2: Identify Risky Directories

Ask the user:

> Which directories contain files that could affect live environments if deployed? (e.g., task files, SQL scripts, pipeline configs, IaC templates, deployment manifests)

Record:
- `RISKY_DIRS` — relative paths to directories containing deployable artifacts
- `RISKY_PATTERNS` — file glob patterns within those directories (e.g., `*.json`, `*.sql`, `*.yaml`)

#### Step 3: Identify Read-Only Operations

Ask the user:

> Do you have operations that target production but are safe (read-only)? For example: query-only SQL, status checks, log pulls.

Record:
- `READONLY_OPS` — operation types or patterns that are safe even when targeting prod

#### Step 4: Identify Protected Branches

Ask the user:

> Which git branches should never be targeted by automated exports or force-pushes?

Default: `main`, `master`. Record as `BLOCKED_BRANCHES`.

#### Step 5: Create safety-rules.json

Write `.claude/safety-rules.json` using the gathered information:

```json
{
  "prodIndicators": ["<PROD_NAMES from step 1>"],
  "prodUriPatterns": ["<PROD_URI_PATTERNS from step 1>"],
  "riskyDirectories": ["<RISKY_DIRS from step 2>"],
  "riskyPatterns": ["<RISKY_PATTERNS from step 2>"],
  "readOnlyOperations": ["<READONLY_OPS from step 3>"],
  "blockedBranches": ["<BLOCKED_BRANCHES from step 4>"]
}
```

#### Step 6: Create safety-gate.sh

Write `.claude/hooks/safety-gate.sh`. This is the deterministic PreToolUse hook. Structure:

```bash
#!/bin/bash
# Production safety gate — deterministic PreToolUse hook for Write|Edit

INPUT=$(cat)
RULES_FILE="$CLAUDE_PROJECT_DIR/.claude/safety-rules.json"

# --- Extract file path from hook input ---
FILE_PATH=$(echo "$INPUT" | node -e "
  let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{
    try{const j=JSON.parse(d);console.log(j.tool_input.file_path||j.tool_input.file||'')}
    catch{console.log('')}
  })" 2>/dev/null)

FILE_PATH=$(echo "$FILE_PATH" | sed 's|\\|/|g')
# Adapt this to extract the relative path for YOUR project:
REL_PATH=$(echo "$FILE_PATH" | sed "s|.*/$(basename $CLAUDE_PROJECT_DIR)/||")

# --- SCOPE CHECK: only inspect risky directories ---
# Replace these patterns with your RISKY_DIRS:
IS_RISKY=false
case "$REL_PATH" in
  # ADD YOUR RISKY DIRECTORY PATTERNS HERE:
  # deploy/*.yaml) IS_RISKY=true ;;
  # terraform/*.tf) IS_RISKY=true ;;
  # tasks/*.json) IS_RISKY=true ;;
  *) ;;
esac

if [ "$IS_RISKY" = false ]; then
  exit 0  # Not a risky file — allow immediately
fi

# --- Extract file content ---
CONTENT=$(echo "$INPUT" | node -e "
  let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{
    try{const j=JSON.parse(d);const ti=j.tool_input;console.log(ti.content||ti.new_string||'')}
    catch{console.log('')}
  })" 2>/dev/null)

# --- Helper functions ---
deny() {
  echo "{\"hookSpecificOutput\":{\"hookEventName\":\"PreToolUse\",\"permissionDecision\":\"deny\",\"permissionDecisionReason\":\"SAFETY GATE: $1\"}}"
  exit 0
}

ask() {
  echo "{\"hookSpecificOutput\":{\"hookEventName\":\"PreToolUse\",\"permissionDecision\":\"ask\",\"permissionDecisionReason\":\"SAFETY CHECK: $1\"}}"
  exit 0
}

# --- Load rules ---
if [ -f "$RULES_FILE" ]; then
  PROD_INDICATORS=$(node -e "const r=require('$RULES_FILE');console.log((r.prodIndicators||[]).join('|'))" 2>/dev/null)
  PROD_URIS=$(node -e "const r=require('$RULES_FILE');console.log((r.prodUriPatterns||[]).join('|'))" 2>/dev/null)
  BLOCKED=$(node -e "const r=require('$RULES_FILE');console.log((r.blockedBranches||[]).join('|'))" 2>/dev/null)
fi

# === ADD YOUR CHECKS HERE ===
# Use 'deny' for things that are ALWAYS wrong (env mismatches, wrong assignments).
# Use 'ask' for things that MIGHT be intentional (deploying to prod).
#
# Pattern: scan $CONTENT for prod indicators
# Example: if the file contains a prod URI under a dev environment label, deny it.
# Example: if the file targets a prod connection, ask for confirmation.
#
# See the full boomerang example at:
# https://github.com/HurleySk/claude-cicd-safety-skill for reference patterns.

# Default: allow
exit 0
```

**Customize the script** for the user's specific project by:
1. Replacing the `case` patterns with their `RISKY_DIRS`
2. Adding checks that scan `$CONTENT` for `PROD_INDICATORS` and `PROD_URIS`
3. Using `deny` for always-wrong patterns (e.g., prod resource under dev config)
4. Using `ask` for intentional-but-risky operations (e.g., deploying to prod)

**Design principles:**
- `deny` = always wrong, no override (environment mismatches, protected branch targeting)
- `ask` = might be intentional, user confirms (prod deployments, prod queries with side effects)
- `exit 0` = safe, no output needed

#### Step 7: Create Bash Confirmation Hook

Write `.claude/hooks/confirm-destructive.sh`:

```bash
#!/bin/bash
INPUT=$(cat)

COMMAND=$(echo "$INPUT" | node -e "
  let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{
    try{console.log(JSON.parse(d).tool_input.command||'')}catch{}
  })" 2>/dev/null)

if echo "$COMMAND" | grep -qE 'git push --force|git push -f'; then
  echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny","permissionDecisionReason":"Force push is not allowed"}}'
elif echo "$COMMAND" | grep -qE 'git push'; then
  echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"ask","permissionDecisionReason":"Confirm: push operation"}}'
else
  exit 0
fi
```

**Optional additions — ask the user before including these:**
- `git merge` confirmation: Useful but may override prior approvals if the user already approved the merge elsewhere. Only add if the user explicitly requests it.
- `git reset --hard` blocking: Prevents destructive resets. Recommended for most projects.

#### Step 8: Wire Up settings.json

Read the user's `.claude/settings.json`. Add hooks to the `PreToolUse` section:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/confirm-destructive.sh\""
          }
        ]
      },
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/safety-gate.sh\""
          },
          {
            "type": "prompt",
            "model": "claude-haiku-4-5-20251001",
            "timeout": 15,
            "prompt": "You MUST respond with ONLY valid JSON: {\"ok\":true} or {\"ok\":false,\"reason\":\"why\"}\n\nEvaluate this file operation: $ARGUMENTS\n\nAllow if path is outside risky directories. Otherwise deny if: production indicators appear under non-production config, deployment targets production without explicit intent, or dangerous operations lack safety guards.\n\nRespond with JSON only."
          }
        ]
      }
    ]
  }
}
```

**Important:** Merge with existing hooks — don't overwrite. If the user already has hooks, add these alongside them.

**Hook ordering matters:** The deterministic script runs first. If it denies, the prompt hook never fires (saves tokens). If it allows, the prompt hook provides a semantic second opinion.

#### Step 9: Test

Generate test inputs and pipe them through the safety-gate.sh script:

```bash
# Test 1: risky file with prod indicator (should deny or ask)
node -e "process.stdout.write(JSON.stringify({tool_input:{
  file_path:'<RISKY_FILE_PATH>',
  content:'<CONTENT_WITH_PROD_INDICATOR>'
}}))" | bash .claude/hooks/safety-gate.sh

# Test 2: non-risky file (should allow — no output, exit 0)
node -e "process.stdout.write(JSON.stringify({tool_input:{
  file_path:'<SAFE_FILE_PATH>',
  content:'safe content'
}}))" | bash .claude/hooks/safety-gate.sh
echo "(exit $?)"

# Test 3: actual Write to a non-risky file (should pass without hook errors)
# Have the user write a test file and confirm no "PreToolUse:Write hook error" appears
```

Create at least one test for each category:
- A file that should be **denied** (always-wrong pattern)
- A file that should trigger **ask** (intentional-but-risky)
- A file that should be **allowed** (safe or out of scope)
- A non-risky file (verifies no false positives)

### `audit`

Review existing safety hooks and report gaps.

#### Step 1: Read Current Configuration

```
Read .claude/settings.json
Glob for .claude/hooks/*.sh
Read .claude/safety-rules.json (if exists)
```

#### Step 2: Analyze Hooks

For each hook found, report:
- **Type**: command, prompt, or agent
- **Matcher**: which tools it intercepts
- **Scope**: what files/operations it checks
- **Decision mode**: deny, ask, or allow

#### Step 3: Gap Analysis

Check for these common gaps:
- [ ] No PreToolUse hook on Write|Edit — files can be written without any safety check
- [ ] No PreToolUse hook on Bash — destructive commands (force push, rm -rf) are unguarded
- [ ] No deterministic checks — relying solely on LLM hooks (slow, may hallucinate)
- [ ] No LLM backstop — relying solely on pattern matching (misses novel issues)
- [ ] No safety-rules.json — prod indicators are hardcoded in scripts (hard to maintain)
- [ ] Using `type: "agent"` instead of `type: "prompt"` — may error in some environments
- [ ] Prompt hook uses wrong response format — must be `{"ok": true/false}`, not `hookSpecificOutput`
- [ ] All risky operations use `deny` — no way to do intentional prod work (should use `ask` for some)
- [ ] No test cases documented — can't verify hooks work after changes

Report findings as a checklist with status and recommendations.

### `help`

**Available commands:**

| Command | Description |
|---------|-------------|
| `/cicd-safety:safety-hooks setup` | Interactive setup — creates safety-rules.json, safety-gate.sh, prompt hook, and wires them into settings.json |
| `/cicd-safety:safety-hooks audit` | Review existing hooks and report safety gaps |
| `/cicd-safety:safety-hooks help` | Show this help |

**Architecture overview:**

```
Write/Edit operation
    │
    ▼
┌─────────────────────┐
│ safety-gate.sh       │  Layer 1: Deterministic pattern matching
│ (type: command)      │  Fast, no tokens, catches known patterns
│                      │  deny = always wrong | ask = confirm
└─────────┬───────────┘
          │ if allowed
          ▼
┌─────────────────────┐
│ prompt hook          │  Layer 2: LLM semantic review
│ (type: prompt)       │  Catches subtle issues script misses
│ Returns ok/not-ok    │  Uses {"ok": true/false} format
└─────────┬───────────┘
          │ if ok
          ▼
       File is written

Bash operation
    │
    ▼
┌─────────────────────┐
│ confirm-destructive  │  Layer 3: Git operation guard
│ (type: command)      │  Blocks force-push, confirms push/merge
└─────────────────────┘
```

## Lessons Learned

These are hard-won lessons from building production safety systems:

1. **Use `ask` not `deny` for intentional prod operations.** If everything is `deny`, users can't do legitimate prod work. Reserve `deny` for things that are ALWAYS wrong (like a prod resource assigned to a dev environment).

2. **`type: "prompt"` hooks return `{"ok": true/false}`.** NOT `{"hookSpecificOutput": {...}}`. Using the wrong format causes "hook error" on every invocation. The `hookSpecificOutput` format is for `type: "command"` hooks only.

3. **`type: "agent"` hooks may error.** In some environments, agent hooks fail silently or timeout. Prefer `type: "prompt"` for LLM-based checks. It's simpler and more reliable.

4. **Put deterministic checks BEFORE LLM checks.** The command hook runs first. If it denies, the LLM never fires — saves tokens and time. If it allows, the LLM provides a second opinion.

5. **Scope hooks tightly.** Don't fire LLM analysis on every file write. Use the command hook to filter: only invoke expensive checks for files in risky directories.

6. **Match environment keywords in names.** If a resource has "prod" in its name, it shouldn't appear under a "dev" config. Simple keyword matching catches most environment mismatches without hardcoded lists.

7. **Use a rules file, not hardcoded patterns.** Keep prod indicators, blocked branches, and risky directories in `safety-rules.json`. When environments change, update one file instead of editing hook scripts.

8. **Test with known-bad inputs.** After setup, pipe test JSON through the hook script to verify deny/ask/allow behavior. Don't trust hooks until you've seen them block something.
