# Example: Vibecheck Assembly Line

Vibecheck is a production Assembly Line running on agentics.dk. It is a fully automated AI code review pipeline — submit any GitHub repository and receive a comprehensive quality report covering architecture, code quality, security, testing, and documentation. This example demonstrates every ALP concept in a real-world context.

## What It Does

A user submits a GitHub repository URL (the Task Card). Eight specialised Claude Code Agents work in sequence, each writing a structured report to a shared stage git repository. The SYNTHESIS Agent aggregates all reports into a scored Vibecheck Report, which is reviewed by a human before the DELIVER Agent publishes it to the Agentics artifact store.

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

Each Agent's `prompt.md` receives `{{task.title}}` and `{{task.description}}` at dispatch time.

---

## Assembly Line Definition

The `.agentics/` folder is split across several files — there is no single monolithic config.

### `.agentics/assembly-line.json`

Top-level metadata only:

```json
{
  "id": "vibecheck",
  "name": "Vibecheck",
  "description": "Comprehensive AI-powered code review pipeline — 8 specialist stations covering intake, architecture, code quality, security, testing, documentation, synthesis, and delivery.",
  "version": "1.0.0"
}
```

### `.agentics/transitions.json`

Defines the pipeline flow. All transitions are `"condition": "success"`. Only the final transition carries a gate:

```json
[
  { "from": "intake",        "to": "architecture",  "condition": "success" },
  { "from": "architecture",  "to": "code-quality",  "condition": "success" },
  { "from": "code-quality",  "to": "security",      "condition": "success" },
  { "from": "security",      "to": "testing",       "condition": "success" },
  { "from": "testing",       "to": "documentation", "condition": "success" },
  { "from": "documentation", "to": "synthesis",     "condition": "success" },
  { "from": "synthesis",     "to": "deliver",       "condition": "success", "gate": "human-review" }
]
```

### `.agentics/gates.json`

```json
[
  {
    "id": "human-review",
    "name": "Human Review",
    "description": "A human reviewer checks the compiled Vibecheck Report (SYNTHESIS output) before the DELIVER agent posts results to GitHub. Approve to publish, reject to stop delivery.",
    "requiresApproval": true
  }
]
```

### Station files

Each station lives in `.agentics/stations/XX-name/` and has three files:

**`station.json`** — trigger config:

```json
{
  "id": "intake",
  "name": "INTAKE",
  "trigger": {
    "type": "dispatch_worker",
    "labels": ["vibecheck"],
    "idleTimeoutMinutes": 5,
    "maxTimeoutMinutes": 25,
    "createStageRepo": true
  }
}
```

`createStageRepo: true` on INTAKE tells the platform to provision a fresh git repository for this run. The URL and credentials are injected as `$STAGE_GIT_URL` and `$STAGE_GIT_TOKEN`. All subsequent stations receive the same env vars automatically.

**`prompt.md`** — the agent's task prompt (supports `{{task.title}}` and `{{task.description}}`):

```markdown
Conduct intake analysis for this Vibecheck submission.

Project: {{task.title}}
{{task.description}}

## Workspace
Your current working directory is **the repository being reviewed** — it was cloned by the
runner before you started. Do not commit or modify files here.

Clone the stage git repository to $STAGE_DIR:
```bash
git -c "http.extraHeader=Authorization: Bearer $STAGE_GIT_TOKEN" clone "$STAGE_GIT_URL" "$STAGE_DIR"
git -C "$STAGE_DIR" config http.extraHeader "Authorization: Bearer $STAGE_GIT_TOKEN"
```

Write your report to $STAGE_DIR/reports/intake.md, then commit and push:
```bash
git -C "$STAGE_DIR" add reports/
git -C "$STAGE_DIR" commit -m "feat(intake): initial analysis"
git -C "$STAGE_DIR" push origin main
```

## Your task
Explore the repository (your CWD) and write reports/intake.md covering:
- Tech stack and versions detected
- Project type and purpose
- Key entry points and architecture overview
- Dependency summary
- Size metrics (file count, LOC by language)
- Any immediate red flags
- Recommended focus areas for downstream stations
```

**`system.md`** — appended to the agent's system prompt to define its role:

```markdown
You are a Technical Intelligence Officer specialising in rapid codebase comprehension.
Your role is to deeply understand a submitted project and produce a precise, structured brief
that downstream agents will rely on. Be factual. Cite specific files and line counts.
If something is unclear or undocumented, say so explicitly — never assume.
Set up the stage workspace, write your brief, commit it, then call stop_broadcast.
```

All 8 stations follow this same three-file pattern. The only differences are `station.json` timeouts and the content of `prompt.md` / `system.md`.

| Station | Role (from system.md) | `maxTimeoutMinutes` |
|---------|----------------------|---------------------|
| INTAKE | Technical Intelligence Officer | 25 |
| ARCHITECTURE | Principal Software Architect | 30 |
| CODE QUALITY | Senior Code Quality Engineer | 30 |
| SECURITY | Application Security Engineer | 30 |
| TESTING | QA Engineer | 30 |
| DOCUMENTATION | Developer Experience Engineer | 20 |
| SYNTHESIS | Lead Reviewer | 20 |
| DELIVER | Delivery Agent | 10 |

---

## Stage Repository Flow

At INTAKE, the platform creates a fresh stage git repository for this run. Every subsequent agent clones it, writes a report, and pushes. The git history is a complete, ordered audit trail of the review.

The runner pre-clones the **submitted repository** into each worker's CWD — agents do not need to fetch the target repo themselves.

```
INTAKE:         CWD = submitted repo (pre-cloned by runner)
                clone stage repo → $STAGE_DIR
                writes $STAGE_DIR/reports/intake.md
                git push

ARCHITECTURE:   CWD = submitted repo
                clone stage repo → $STAGE_DIR
                reads  $STAGE_DIR/reports/intake.md
                writes $STAGE_DIR/reports/architecture.md
                git push

SECURITY:       CWD = submitted repo
                clone stage repo → $STAGE_DIR
                reads  $STAGE_DIR/reports/intake.md + architecture.md
                writes $STAGE_DIR/reports/security.md
                git push

... (CODE QUALITY, TESTING, DOCUMENTATION) ...

SYNTHESIS:      clone stage repo → $STAGE_DIR
                reads  $STAGE_DIR/reports/*.md
                writes $STAGE_DIR/vibecheck-report.md
                git push

DELIVER:        clone stage repo → $STAGE_DIR
                reads  $STAGE_DIR/vibecheck-report.md
                POSTs  artifact to Agentics API
                writes $STAGE_DIR/reports/delivery-result.md
                git push
```

---

## Vibe Score

SYNTHESIS produces a weighted composite score across five dimensions:

| Dimension     | Weight |
|---------------|--------|
| Security      | 30%    |
| Code Quality  | 25%    |
| Architecture  | 20%    |
| Testing       | 15%    |
| Documentation | 10%    |

Each dimension is scored 0–10. A total Vibe Score below **6.0 / 10** is a failing grade.

---

## DELIVER: Agentics Artifact Store

DELIVER does not post a GitHub comment. It POSTs a structured artifact to the Agentics REST API:

```
POST ${AGENTICS_BASE_URL}/api/owners/${AGENTICS_OWNER}/projects/${AGENTICS_PROJECT_NAME}/artifacts
Authorization: Bearer $AGENTICS_TOKEN
Content-Type: application/json

{
  "type": "vibecheck",
  "name": "Vibecheck — <project>",
  "data": {
    "vibeScore": 7.4,
    "categories": {
      "security":      { "score": 8 },
      "codeQuality":   { "score": 7 },
      "architecture":  { "score": 7 },
      "testing":       { "score": 6 },
      "documentation": { "score": 9 }
    },
    "reportMarkdown": "..."
  }
}
```

The response includes an `id` used to construct the public artifact URL:
```
https://agentics.dk/project/{owner}/{project}/artifacts/{id}
```

This page displays the Vibe Score as a diploma-style number with per-category score bars and the full report in a collapsible section.

---

## Plugin Skills

The repo ships a reusable skill at `plugin/skills/deliver-vibecheck/SKILL.md`. This is an example of how assembly lines can bundle custom skills that agents load at runtime — the DELIVER station's `prompt.md` references it directly:

> *Follow the `deliver-vibecheck` skill (see plugin/skills/deliver-vibecheck/SKILL.md)*

This pattern keeps complex multi-step instructions out of the prompt and makes them reusable across assembly lines.

---

## ALP Concepts in This Example

| Concept | In Vibecheck |
|---|---|
| Assembly Line | The 8-station Vibecheck pipeline |
| Stations | INTAKE, ARCHITECTURE, CODE QUALITY, SECURITY, TESTING, DOCUMENTATION, SYNTHESIS, DELIVER |
| Task Card | GitHub repo URL + description in task description field |
| Agent Definition | `prompt.md` + `system.md` per station, rendered with `{{task.title}}` / `{{task.description}}` |
| Stage Repository | Created at INTAKE (`createStageRepo: true`), shared across all 8 stations via `$STAGE_GIT_URL` / `$STAGE_GIT_TOKEN` / `$STAGE_DIR` |
| Transition Rules | `transitions.json` — 7 rules, all `condition: success` |
| Gate | `human-review` gate blocks SYNTHESIS → DELIVER; human approves before artifact is published |
| Task lifecycle | queued → in_progress (×8) → awaiting_review → completed |
| Labels | `["vibecheck"]` — any worker registered with the vibecheck label |
| Plugin Skills | `plugin/skills/deliver-vibecheck/` — reusable delivery skill bundled with the assembly line |
