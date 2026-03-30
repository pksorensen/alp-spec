# 03 — Assembly Lines

An **Assembly Line** is an ordered sequence of Stations that defines how a type of work is processed end-to-end. It is the top-level concept in ALP — everything else (Stations, Tasks, Jobs, Transition Rules) exists in the context of an Assembly Line.

---

## Structure

An Assembly Line consists of:

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique identifier (assigned by Server) |
| `name` | string | yes | Human-readable name (e.g. "Vibecheck", "PR Review") |
| `description` | string | no | What this pipeline does |
| `version` | string | no | Semantic version (e.g. "1.0.0") |
| `stations` | Station[] | yes | Ordered list of Stations (see [04-stations.md](04-stations.md)) |
| `transitionRules` | TransitionRule[] | no | Flow control rules (see [06-transition-rules.md](06-transition-rules.md)) |
| `gates` | Gate[] | no | Human review gate definitions |
| `owner` | string | yes | Account that owns this Assembly Line |
| `project` | string | yes | Project this Assembly Line belongs to |

---

## Example

```json
{
  "id": "vibecheck",
  "name": "Vibecheck",
  "description": "Comprehensive AI-powered code review pipeline",
  "version": "1.0.0",
  "owner": "pksorensen",
  "project": "agentics",
  "stations": [
    { "id": "intake",       "name": "INTAKE",        "trigger": { "type": "dispatch_worker", "labels": ["vibecheck"] } },
    { "id": "architecture", "name": "ARCHITECTURE",  "trigger": { "type": "dispatch_worker", "labels": ["vibecheck"] } },
    { "id": "security",     "name": "SECURITY",      "trigger": { "type": "dispatch_worker", "labels": ["vibecheck"] } },
    { "id": "synthesis",    "name": "SYNTHESIS",     "trigger": { "type": "dispatch_worker", "labels": ["vibecheck"] } },
    { "id": "deliver",      "name": "DELIVER",       "trigger": { "type": "dispatch_worker", "labels": ["vibecheck"] } }
  ],
  "transitionRules": [
    { "id": "r1", "fromStationId": "synthesis", "toStationId": "deliver", "condition": "success", "gateId": "human-review" }
  ],
  "gates": [
    { "id": "human-review", "name": "Human Review", "requiresApproval": true }
  ]
}
```

---

## Assembly Line vs Task

An Assembly Line is a **template** (the definition). A **Task** is a **run instance** of that template. Many Tasks can be in flight on the same Assembly Line simultaneously, each at a different Station.

```
Assembly Line: Vibecheck
  ├── Task abc (at SECURITY Station, running)
  ├── Task def (at SYNTHESIS Station, awaiting review)
  └── Task ghi (just submitted, queued at INTAKE)
```

---

## Assembly Line Repository

Every Task that runs through an Assembly Line gets its own **Assembly Line Repository** — a hosted git repository provisioned by the Server. This repository is the shared workspace for all Agents working on that Task across all Stations.

The Assembly Line Repository:
- Is created automatically when a Task reaches the Station configured with `createAssemblyLineRepo: true` (typically the first Station)
- Has a unique URL and Bearer token (provided to each Agent via `ASSEMBLY_LINE_REPO_URL` and `ASSEMBLY_LINE_REPO_TOKEN` environment variables)
- Persists the full history of everything every Agent produced — reports, generated code, logs
- Is read-only accessible to reviewers and the task owner via the Server UI

See [spec/10-agent.md](10-agent.md) for how Agents use the repository.

---

## Linearity Constraint (v1)

**Current version:** Assembly Lines are strictly linear — one path from the first Station to the last. A Task enters Station 1, completes it, moves to Station 2, and so on. There is no branching and no parallelism in v1.

> **Note:** Linearity is a v1 constraint, not a permanent design choice. The Transition Rule schema is designed to accommodate `goto` rules that enable conditional routing in a future version. See [spec/06-transition-rules.md](06-transition-rules.md) for the forward-looking design.

---

## Default Transition Behavior

When an Assembly Line has no Transition Rules defined, the Server uses the following defaults:

| Outcome | Behavior |
|---|---|
| Station succeeds (not last) | Task advances to the next Station |
| Station succeeds (last Station) | Task → `completed` |
| Station fails | Task → `failed`, pipeline stops |

Transition Rules override this default on a per-station basis. See [06-transition-rules.md](06-transition-rules.md).
