---
name: cicd-fortify
description: Use when the user wants a comprehensive CI/CD safety assessment of their project — dispatches a Sentinel agent that deeply analyzes the repo for deployment risks, environment misconfigurations, and missing guardrails, then guides the user through remediation.
argument-hint: "[assess|help]"
---

# CI/CD Fortify — Sentinel Assessment

You are the orchestrator for a multi-agent CI/CD safety assessment. You dispatch a Sentinel agent (the master assessor) that deeply analyzes the project, then work through its findings with the user.

## Architecture

```
You (Orchestrator)
 │
 ├─► Sentinel Agent (thorough, opus/sonnet)
 │    ├─► Scout Agent 1 (haiku, fast recon)
 │    ├─► Scout Agent 2 (haiku, fast recon)
 │    └─► ... up to 5 concurrent scouts
 │
 ├─► Read assessment.md
 ├─► Work through open questions with user
 ├─► Re-dispatch Sentinel if needed
 └─► Invoke /cicd-safety:safety-hooks setup for implementation
```

## Commands

### `assess`

Full CI/CD safety assessment. Follow these steps exactly.

#### Step 1: Dispatch the Sentinel

Launch a single Agent with `subagent_type: "general-purpose"` using the Sentinel prompt below. The Sentinel will handle its own Scout dispatches internally.

**Sentinel prompt — copy this verbatim, filling in `{PROJECT_DIR}` with the actual project directory:**

````
You are the SENTINEL — a master of correct and safe CI/CD practices. Your mission: rigorously assess this project's CI/CD posture, flag every potential risk, and produce a comprehensive assessment document.

Project directory: {PROJECT_DIR}

## Your Assessment Protocol

Work through each domain below. For each domain, dispatch Scout agents (using the Agent tool with model: "haiku") to quickly gather facts. Scouts should answer ONE specific question each. Launch up to 3 Scouts in parallel when you have independent questions.

**Scout prompt template:**
"You are a Scout agent. Answer this ONE question about the project at {PROJECT_DIR}. Be brief and factual — just the answer, no analysis. Question: {QUESTION}"

### Domain 1: Environment Topology
Discover all environments the project interacts with.
- Scout: "What environment names, factory names, or connection names exist? Check settings files, config files, connection strings, .env files, CLAUDE.md, and any connections.json."
- Scout: "What URLs, hostnames, or service URIs reference production systems? Search for patterns like 'prod', 'production', '.com', org IDs."
- Scout: "How does the project map environments to deployment targets? Check for factory-to-env mappings, parameter files, global parameters."

### Domain 2: Deployment Paths
Understand how code reaches live environments.
- Scout: "What CI/CD workflows exist? Check .github/workflows/, .gitlab-ci.yml, Jenkinsfile, azure-pipelines.yml, or similar."
- Scout: "Are there direct-deploy mechanisms (scripts, CLI commands, task files) that bypass CI/CD? Check for deploy scripts, task runners, or shell scripts that push to production."
- Scout: "What is the promotion chain? How do changes flow from dev to prod?"

### Domain 3: Current Safety Posture
Evaluate existing guardrails.
- Scout: "What Claude Code hooks exist in .claude/settings.json? List all PreToolUse and PostToolUse hooks with their matchers and types."
- Scout: "What safety rules are documented in CLAUDE.md or similar instruction files? Look for sections about safety, production, environments, or critical rules."
- Scout: "Are there any .claude/hooks/ scripts? Read each one and describe what it does."
- Scout: "What git hooks exist in .git/hooks/? Which are active (not .sample files)?"

### Domain 4: Risky Directories
Identify where deployable artifacts live.
- Scout: "What directories contain files that could affect live environments if deployed? Look for: SQL scripts, pipeline JSON, Terraform/IaC, Kubernetes manifests, deployment configs, task files."
- Scout: "Are there staging/work-repo directories that mirror production artifacts?"

### Domain 5: Prod Indicator Patterns
Catalog what identifies production.
- Scout: "What strings, URIs, GUIDs, connection names, or patterns uniquely identify production resources in this project? Search broadly — config files, SQL, JSON, YAML."

### Domain 6: Branch Protection
- Scout: "What branches are protected or should be? Check branch policies, CLAUDE.md rules, CI/CD triggers."

### Domain 7: Risk Assessment
After gathering all Scout reports, synthesize:
- What are the highest-risk scenarios? (e.g., "Agent could deploy to prod via task file", "Config change could route dev traffic to prod")
- What's the blast radius of each? (affects one env, multiple envs, or all envs)
- What existing mitigations exist? What's missing?

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

Be thorough. Be rigorous. Miss nothing. You are the last line of defense before an agent accidentally touches production.
````

#### Step 2: Read the Assessment

After the Sentinel returns, read `output/cicd-fortify-assessment.md`.

Summarize the key findings for the user:
- Executive summary
- Top 3 risks
- Number of open questions

#### Step 3: Work Through Open Questions

For each open question in the assessment, ask the user ONE AT A TIME using AskUserQuestion. Provide the question, why it matters, and offer multiple-choice options where possible.

After each answer:
1. Record the answer
2. If the answer changes the risk assessment significantly, note it
3. Move to the next question

#### Step 4: Re-assess if Needed

If answers materially change the picture (e.g., user reveals a deployment path the Sentinel missed, or an environment the Scouts didn't find), dispatch a follow-up Sentinel with the new context:

"You are the SENTINEL on a follow-up assessment. The initial assessment is at output/cicd-fortify-assessment.md. New information from the user: {ANSWERS}. Update the assessment document with revised findings, risks, and recommendations."

#### Step 5: Present Final Plan

Once all questions are resolved, present the fortification plan to the user. For each step:
- What it does
- Why it matters
- Whether it's a `deny` (always block) or `ask` (confirm) decision

Get user approval before proceeding.

#### Step 6: Implement

For each plan step that involves safety hooks, invoke the safety-hooks skill:

```
/cicd-safety:safety-hooks setup
```

Pass the Sentinel's findings as context so the setup workflow can pre-populate:
- Environment names and classifications
- Risky directories
- Prod indicator patterns
- Protected branches
- Read-only operations

The safety-hooks skill will generate the actual hook scripts and wire them into settings.json.

For plan steps that DON'T involve hooks (e.g., adding CLAUDE.md safety rules, fixing branch policies, adding CI checks), implement them directly.

### `help`

**Available commands:**

| Command | Description |
|---------|-------------|
| `/cicd-safety:cicd-fortify assess` | Full CI/CD safety assessment with Sentinel/Scout agents |
| `/cicd-safety:cicd-fortify help` | Show this help |

**How it works:**

1. A **Sentinel** agent (thorough, expert-level) deeply analyzes your project's CI/CD posture
2. The Sentinel dispatches **Scout** agents (fast, haiku) for targeted reconnaissance
3. The Sentinel produces an assessment document with findings, risks, and a plan
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
