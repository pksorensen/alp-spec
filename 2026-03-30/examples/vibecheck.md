# Example: Vibecheck Assembly Line

Vibecheck is a production Assembly Line running on agentics.dk. It is a fully automated AI code review pipeline — submit any GitHub repository and receive a comprehensive quality report covering architecture, code quality, security, testing, and documentation. This example demonstrates every ALP concept in a real-world context.

---

## What It Does

A user submits a GitHub repository URL (the Task Card). Eight specialised Claude Code Agents work in sequence, each writing a structured report to the Assembly Line Repository. The final Agent synthesises all reports into a scored Vibecheck Report and delivers it as a GitHub comment.

```
INTAKE → ARCHITECTURE → CODE QUALITY → SECURITY → TESTING → DOCUMENTATION
                                                                    ↓
DELIVER ← [Gate: HUMAN REVIEW] ← SYNTHESIS ─────────────────────────
```

---

## Task Card

```
Title:   Vibecheck: my-startup/backend

Description:
  Repository:  https://github.com/my-startup/backend
  Branch:      main
  Stack:       Node.js + PostgreSQL   (optional — agent auto-detects)
  Context:     About to open-source this. Want a thorough review first.
  Concerns:    Auth implementation, SQL injection risks
```

The Agent at each Station reads `{{task.description}}` via its prompt template and knows exactly what to review.

---

## Assembly Line Definition

```json
{
  "id": "vibecheck",
  "name": "Vibecheck",
  "description": "Comprehensive AI-powered code review pipeline",
  "version": "1.0.0",
  "stations": [
    {
      "id": "intake",
      "name": "INTAKE",
      "trigger": {
        "type": "dispatch_worker",
        "labels": ["vibecheck", "claude-code"],
        "createAssemblyLineRepo": true,
        "promptTemplate": "You are the INTAKE agent for the Vibecheck pipeline.\n\nTask: {{task.title}}\n\n{{task.description}}\n\nSetup the workspace:\n1. Clone the Assembly Line Repository: git clone $ASSEMBLY_LINE_REPO_URL workspace && cd workspace\n2. Create the target/ directory and clone the submitted repository into it\n3. Write workspace/reports/intake.md with: repository URL, detected stack, directory structure summary\n4. Commit and push\n\nExit when done."
      }
    },
    {
      "id": "architecture",
      "name": "ARCHITECTURE",
      "trigger": {
        "type": "dispatch_worker",
        "labels": ["vibecheck", "claude-code"],
        "promptTemplate": "You are the ARCHITECTURE agent for the Vibecheck pipeline.\n\nTask: {{task.title}}\n\n{{task.description}}\n\n1. Clone the Assembly Line Repository: git clone $ASSEMBLY_LINE_REPO_URL workspace && cd workspace\n2. Read workspace/reports/intake.md\n3. Analyse the architecture of the codebase in workspace/target/\n4. Write workspace/reports/architecture.md with: system design analysis, component relationships, architectural concerns\n5. Commit and push\n\nExit when done."
      }
    },
    {
      "id": "code-quality",
      "name": "CODE QUALITY",
      "trigger": {
        "type": "dispatch_worker",
        "labels": ["vibecheck", "claude-code"],
        "promptTemplate": "You are the CODE QUALITY agent for the Vibecheck pipeline.\n\nTask: {{task.title}}\n\n{{task.description}}\n\n1. Clone the Assembly Line Repository\n2. Analyse code quality: patterns, complexity, naming, dead code\n3. Write reports/code-quality.md\n4. Commit and push"
      }
    },
    {
      "id": "security",
      "name": "SECURITY",
      "trigger": {
        "type": "dispatch_worker",
        "labels": ["vibecheck", "claude-code"],
        "promptTemplate": "You are the SECURITY agent for the Vibecheck pipeline.\n\nTask: {{task.title}}\n\n{{task.description}}\n\n1. Clone the Assembly Line Repository\n2. Audit for: auth vulnerabilities, injection risks, secrets in code, dependency CVEs\n3. Write reports/security.md with severity ratings (CRITICAL/HIGH/MEDIUM/LOW)\n4. Commit and push"
      }
    },
    {
      "id": "testing",
      "name": "TESTING",
      "trigger": {
        "type": "dispatch_worker",
        "labels": ["vibecheck", "claude-code"],
        "promptTemplate": "You are the TESTING agent for the Vibecheck pipeline.\n\nTask: {{task.title}}\n\n{{task.description}}\n\n1. Clone the Assembly Line Repository: git clone $ASSEMBLY_LINE_REPO_URL workspace && cd workspace\n2. Analyse the test suite in workspace/target/: coverage, test quality, missing tests, flaky tests\n3. Write reports/testing.md with: test framework(s) found, estimated coverage, critical untested paths, recommendations\n4. Commit and push\n\nExit when done."
      }
    },
    {
      "id": "documentation",
      "name": "DOCUMENTATION",
      "trigger": {
        "type": "dispatch_worker",
        "labels": ["vibecheck", "claude-code"],
        "promptTemplate": "You are the DOCUMENTATION agent for the Vibecheck pipeline.\n\nTask: {{task.title}}\n\n{{task.description}}\n\n1. Clone the Assembly Line Repository: git clone $ASSEMBLY_LINE_REPO_URL workspace && cd workspace\n2. Evaluate documentation quality in workspace/target/: README, inline comments, API docs, changelogs\n3. Write reports/documentation.md with: documentation coverage score, missing critical docs, quality assessment\n4. Commit and push\n\nExit when done."
      }
    },
    {
      "id": "synthesis",
      "name": "SYNTHESIS",
      "trigger": {
        "type": "dispatch_worker",
        "labels": ["vibecheck", "claude-code"],
        "promptTemplate": "You are the SYNTHESIS agent for the Vibecheck pipeline.\n\n1. Clone the Assembly Line Repository\n2. Read ALL reports in reports/\n3. Write vibecheck-report.md: scored summary (0–10 per category), top findings, recommendations\n4. Commit and push\n\nThis report will be reviewed by a human before delivery."
      }
    },
    {
      "id": "deliver",
      "name": "DELIVER",
      "trigger": {
        "type": "dispatch_worker",
        "labels": ["vibecheck", "claude-code"],
        "promptTemplate": "You are the DELIVER agent for the Vibecheck pipeline.\n\n1. Clone the Assembly Line Repository\n2. Read vibecheck-report.md\n3. Post it as a GitHub comment on the submitted repository's default branch\n4. Use the GitHub API with the credentials available in your environment"
      }
    }
  ],
  "transitionRules": [
    {
      "id": "synthesis-to-deliver",
      "fromStationId": "synthesis",
      "toStationId": "deliver",
      "condition": "success",
      "gateId": "human-review"
    }
  ],
  "gates": [
    {
      "id": "human-review",
      "name": "Human Review",
      "description": "A human reviewer checks the Vibecheck report before it is delivered to the submitter.",
      "requiresApproval": true
    }
  ]
}
```

---

## Assembly Line Repository Flow

```
INTAKE:         git clone repo → target/
                writes reports/intake.md
                git push

ARCHITECTURE:   git clone ALR
                reads  reports/intake.md
                writes reports/architecture.md
                git push

SECURITY:       git clone ALR
                reads  reports/intake.md + architecture.md
                writes reports/security.md
                git push

... (TESTING, DOCUMENTATION) ...

SYNTHESIS:      git clone ALR
                reads  reports/*.md
                writes vibecheck-report.md
                git push

DELIVER:        git clone ALR
                reads  vibecheck-report.md
                posts  GitHub comment
```

Each Agent works independently but builds on the work of previous Agents via the shared Assembly Line Repository (ALR). The git history is the complete audit trail.

---

## ALP Concepts in This Example

| Concept | In Vibecheck |
|---|---|
| Assembly Line | The 8-station Vibecheck pipeline |
| Stations | INTAKE, ARCHITECTURE, SECURITY, etc. |
| Task Card | GitHub repo URL + description in task description field |
| Agent Definition | `promptTemplate` per Station, rendered with task data |
| Assembly Line Repository | Created at INTAKE, shared across all 8 Stations |
| Transition Rule | SYNTHESIS → DELIVER blocked by HUMAN_REVIEW gate |
| Gate | Human approves the report before DELIVER sends it |
| Task lifecycle | queued → in_progress (×8) → awaiting_review → completed |
| Labels | `["vibecheck", "claude-code"]` — any Claude Code client with the vibecheck label |
| Default transitions | All other Station-to-Station flows use the default (success → next) |
