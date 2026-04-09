---
name: cicd-fortify
description: Use when the user wants a comprehensive CI/CD safety assessment of their project — dispatches Scout and Sentinel agents to analyze the repo for deployment risks, environment misconfigurations, and missing guardrails, then guides the user through remediation.
argument-hint: "[assess|help]"
---

# CI/CD Fortify — Sentinel Assessment

You are the orchestrator for a multi-agent CI/CD safety assessment. You dispatch Scout agents (haiku, fast recon) in parallel to gather facts, then pass their findings to a Sentinel agent (the master assessor) for synthesis. Finally, you work through the Sentinel's findings with the user.

## Architecture

```
You (Orchestrator)
 │
 ├─► Scout 1 (haiku) — Environment Topology
 ├─► Scout 2 (haiku) — Deployment Paths & Safety Posture
 ├─► Scout 3 (haiku) — Risky Dirs, Prod Indicators, Branch Protection
 │   (all 3 launched in parallel via Agent tool)
 │
 ├─► Sentinel Agent (inherits model) — receives Scout findings
 │   Synthesizes assessment, writes output files
 │
 ├─► Read assessment.md
 ├─► Work through open questions with user
 ├─► Re-dispatch Sentinel if needed
 └─► Invoke /cicd-safety:safety-hooks setup for implementation
```

## Commands

### `assess`

Full CI/CD safety assessment. Follow these steps exactly.

#### Step 1: Dispatch Scouts

Launch **3 Scout agents in parallel** (all in a single message with 3 Agent tool calls). Each Scout is a haiku agent that gathers facts for its assigned domains. Scouts should be fast, factual, and exhaustive — no analysis, just findings.

Fill in `{PROJECT_DIR}` with the actual project directory in each prompt.

**Scout 1 — Environment Topology:**

```
subagent_type: "general-purpose"
model: "haiku"
```

Prompt:
````
You are a Scout agent performing fast reconnaissance on the project at {PROJECT_DIR}.

Answer these questions. Be brief and factual — list what you find, no analysis.

1. What environment names, factory names, or connection names exist? Check settings files, config files, connection strings, .env files, CLAUDE.md, and any connections.json.
2. What URLs, hostnames, or service URIs reference production systems? Search for patterns like 'prod', 'production', '.com', org IDs.
3. How does the project map environments to deployment targets? Check for factory-to-env mappings, parameter files, global parameters.

Format your response as:
## Environment Topology Findings
(your findings, organized by question)
````

**Scout 2 — Deployment Paths & Safety Posture:**

```
subagent_type: "general-purpose"
model: "haiku"
```

Prompt:
````
You are a Scout agent performing fast reconnaissance on the project at {PROJECT_DIR}.

Answer these questions. Be brief and factual — list what you find, no analysis.

1. What CI/CD workflows exist? Check .github/workflows/, .gitlab-ci.yml, Jenkinsfile, azure-pipelines.yml, or similar.
2. Are there direct-deploy mechanisms (scripts, CLI commands, task files) that bypass CI/CD? Check for deploy scripts, task runners, or shell scripts that push to production.
3. What is the promotion chain? How do changes flow from dev to prod?
4. What Claude Code hooks exist in .claude/settings.json? List all PreToolUse and PostToolUse hooks with their matchers and types.
5. What safety rules are documented in CLAUDE.md or similar instruction files? Look for sections about safety, production, environments, or critical rules.
6. Are there any .claude/hooks/ scripts? Read each one and describe what it does.
7. What git hooks exist in .git/hooks/? Which are active (not .sample files)?

Format your response as:
## Deployment Paths Findings
(findings for questions 1-3)

## Safety Posture Findings
(findings for questions 4-7)
````

**Scout 3 — Risky Directories, Prod Indicators & Branch Protection:**

```
subagent_type: "general-purpose"
model: "haiku"
```

Prompt:
````
You are a Scout agent performing fast reconnaissance on the project at {PROJECT_DIR}.

Answer these questions. Be brief and factual — list what you find, no analysis.

1. What directories contain files that could affect live environments if deployed? Look for: SQL scripts, pipeline JSON, Terraform/IaC, Kubernetes manifests, deployment configs, task files.
2. Are there staging/work-repo directories that mirror production artifacts?
3. What strings, URIs, GUIDs, connection names, or patterns uniquely identify production resources in this project? Search broadly — config files, SQL, JSON, YAML.
4. What branches are protected or should be? Check branch policies, CLAUDE.md rules, CI/CD triggers.

Format your response as:
## Risky Directories Findings
(findings for questions 1-2)

## Prod Indicator Findings
(findings for question 3)

## Branch Protection Findings
(findings for question 4)
````

#### Step 2: Dispatch the Sentinel

After all 3 Scouts return, collect their findings and launch the Sentinel agent. The Sentinel synthesizes Scout findings into a comprehensive assessment.

Launch a single Agent using the Sentinel prompt below. Fill in `{PROJECT_DIR}` and `{SCOUT_FINDINGS}` (paste the combined output from all 3 Scouts).

```
subagent_type: "general-purpose"
```

(No `model` parameter — the Sentinel inherits the dispatcher's model for maximum quality.)

**Sentinel prompt:**

````
You are the SENTINEL — a master of correct and safe CI/CD practices. Your mission: synthesize reconnaissance findings into a comprehensive CI/CD safety assessment.

Project directory: {PROJECT_DIR}

## Scout Findings

The following findings were gathered by Scout agents. Use these as your primary data source. You may perform additional investigation if the Scout findings are incomplete or raise follow-up questions, but do not redo work the Scouts already completed.

{SCOUT_FINDINGS}

## Your Assessment Protocol

Using the Scout findings above, perform the following synthesis:

### Risk Assessment
- What are the highest-risk scenarios? (e.g., "Agent could deploy to prod via task file", "Config change could route dev traffic to prod")
- What's the blast radius of each? (affects one env, multiple envs, or all envs)
- What existing mitigations exist? What's missing?

### Gap Analysis
- Cross-reference deployment paths with safety posture — are all paths covered?
- Cross-reference prod indicators with hook rules — are all prod resources protected?
- Identify any environments, paths, or artifacts that lack guardrails.

## Your Output

Write a file to `output/cicd-fortify-assessment.md` with this exact structure:

```markdown
# CI/CD Fortification Assessment

**Project:** {project name}
**Date:** {date}
**Sentinel:** Claude Code automated assessment

## Executive Summary
2-3 sentences: overall safety posture, most critical finding, recommended priority.

## Environment Topology
| Environment | Identifiers | Classification |
|-------------|-------------|----------------|
(table of all discovered environments with names, URIs, and dev/staging/prod classification)

## Deployment Paths
For each path: source → mechanism → target. Flag any that bypass CI/CD.

## Current Safety Posture

### Existing Guardrails
- (list what's already in place with effectiveness rating: strong/moderate/weak)

### Missing Guardrails
- (list what's missing, ordered by risk)

## Risk Assessment
| # | Risk | Severity | Blast Radius | Current Mitigation | Recommendation |
|---|------|----------|--------------|-------------------|----------------|
(ranked table, severity: critical/high/medium/low)

## Recommended Fortification Plan
Numbered steps in priority order. For each step that involves setting up safety hooks, note: "→ Implement via /cicd-safety:safety-hooks setup"

## Open Questions
Items the Sentinel could not determine from the codebase alone. Each question includes:
- The question
- Why it matters for the assessment
- What the Sentinel would do differently depending on the answer

## Scout Reports
(Raw findings from each Scout, for reference)
```

**ALSO** write a machine-readable rules file to `output/cicd-fortify-rules.json` using this schema:

```json
{
  "prodIndicators": ["<all discovered prod environment names, factory names, connection names>"],
  "prodUriPatterns": ["<all discovered production URIs, hostnames, org IDs>"],
  "riskyDirectories": ["<all directories containing deployable artifacts>"],
  "riskyPatterns": ["<file glob patterns within risky directories>"],
  "readOnlyOperations": ["<safe operations that target prod: read-only SQL, status checks>"],
  "blockedBranches": ["<branches that should not be targeted by automated pushes>"]
}
```

This file will be consumed by the `safety-hooks setup` workflow to pre-populate hook configuration. Include only values you are confident about — leave arrays empty rather than guessing.

Be thorough. Be rigorous. Miss nothing. You are the last line of defense before an agent accidentally touches production.
````

#### Step 3: Read the Assessment

After the Sentinel returns, read `output/cicd-fortify-assessment.md`.

Summarize the key findings for the user:
- Executive summary
- Top 3 risks
- Number of open questions

#### Step 4: Work Through Open Questions

For each open question in the assessment, ask the user ONE AT A TIME using AskUserQuestion. Provide the question, why it matters, and offer multiple-choice options where possible.

After each answer:
1. Record the answer
2. If the answer changes the risk assessment significantly, note it
3. Move to the next question

#### Step 5: Re-assess if Needed

If answers materially change the picture (e.g., user reveals a deployment path the Sentinel missed, or an environment the Scouts didn't find), dispatch a follow-up Sentinel with the new context:

"You are the SENTINEL on a follow-up assessment. The initial assessment is at output/cicd-fortify-assessment.md. New information from the user: {ANSWERS}. Update the assessment document with revised findings, risks, and recommendations."

#### Step 6: Present Final Plan

Once all questions are resolved, present the fortification plan to the user. For each step:
- What it does
- Why it matters
- Whether it's a `deny` (always block) or `ask` (confirm) decision

Get user approval before proceeding.

#### Step 7: Implement

For each plan step that involves safety hooks, invoke the safety-hooks skill:

```
/cicd-safety:safety-hooks setup
```

The Sentinel wrote `output/cicd-fortify-rules.json` — pass this file path to the safety-hooks setup so it can pre-populate the configuration. Read the file and use its values to answer the setup wizard's questions (Steps 1-4) without requiring the user to re-enter information the Sentinel already discovered.

The safety-hooks skill will generate the actual hook scripts and wire them into settings.json.

For plan steps that DON'T involve hooks (e.g., adding CLAUDE.md safety rules, fixing branch policies, adding CI checks), implement them directly.

### `help`

**Available commands:**

| Command | Description |
|---------|-------------|
| `/cicd-safety:cicd-fortify assess` | Full CI/CD safety assessment with Sentinel/Scout agents |
| `/cicd-safety:cicd-fortify help` | Show this help |

**How it works:**

1. Three **Scout** agents (fast, haiku) are dispatched in parallel for targeted reconnaissance across all domains
2. A **Sentinel** agent (thorough, expert-level) receives Scout findings and synthesizes a comprehensive assessment
3. The Sentinel produces an assessment document with findings, risks, and a fortification plan
4. You work through open questions with the orchestrator
5. The plan is implemented using the `safety-hooks` skill

**What the Sentinel assesses:**
- Environment topology (what environments exist, how they're identified)
- Deployment paths (how changes reach production)
- Current safety hooks and guardrails
- Risky directories (where deployable artifacts live)
- Production indicators (URIs, connection strings, env names)
- Branch protection
- Risk scenarios with blast radius analysis
