# 04 — Stations

A **Station** is one discrete unit of work in an Assembly Line. When a Task reaches a Station, the Server dispatches a Job to a matching Runner. The Runner spawns a Station Operator, which runs an Agent using the Station's configuration.

---

## Structure

A Station is defined by its **trigger** — the configuration that determines how the Job is dispatched and what the Agent receives.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique identifier within the Assembly Line (e.g. `"security"`) |
| `name` | string | yes | Display name (e.g. `"SECURITY"`) |
| `trigger` | StationTrigger | yes | Dispatch configuration (see below) |

---

## Station Trigger

The trigger defines whether and how a Job is dispatched when a Task arrives at this Station.

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `"dispatch_worker"` \| `"none"` | `"none"` | `dispatch_worker` creates a Job and sends it to a Runner; `none` is a manual or non-automated station |
| `labels` | string[] | `[]` | Required Runner labels for this Station (see Label Matching below) |
| `promptTemplate` | string | `""` | The Agent's instruction. Supports `{{task.title}}` and `{{task.description}}` interpolation |
| `appendSystemPrompt` | string | `""` | Additional text appended to the Agent's system prompt |
| `idleTimeoutMinutes` | number | `2` | Cancel the Job if the Agent is idle for this many minutes |
| `maxTimeoutMinutes` | number | `60` | Hard wall-clock limit for the entire Station execution |
| `initBranch` | boolean | `false` | If true, the Runner creates a git branch named `task-{taskId}` in the repository before starting the Agent |
| `autoGit` | boolean | `false` | If true, the Agent's session is blocked from ending if there are uncommitted changes |
| `commitMessageTemplate` | string | `""` | Hint injected into the Agent's context when `autoGit` blocks |
| `createAssemblyLineRepo` | boolean | `false` | Provision an Assembly Line Repository for this Task when it arrives here. Enable on the first Station that needs it. |

---

## Example Station Definition

```json
{
  "id": "security",
  "name": "SECURITY",
  "trigger": {
    "type": "dispatch_worker",
    "labels": ["vibecheck", "claude-code"],
    "promptTemplate": "You are a security expert reviewing a codebase.\n\nTask: {{task.title}}\n\n{{task.description}}\n\nClone the Assembly Line Repository:\n  git clone $ASSEMBLY_LINE_REPO_URL workspace\n  cd workspace\n\nThe target repository is in ./target/\n\nPerform a thorough security audit. Write your findings to reports/security.md.\nCommit and push when done.",
    "appendSystemPrompt": "You are working as part of an automated code review pipeline. Write structured, actionable reports. Always commit and push your outputs to the Assembly Line Repository before exiting.",
    "idleTimeoutMinutes": 30,
    "maxTimeoutMinutes": 60,
    "createAssemblyLineRepo": false
  }
}
```

---

## Agent Definition (the Job payload)

When the Server dispatches a Job for a Station, it builds an **Agent Definition** and sends it to the Runner. The Agent Definition is derived from the Station trigger plus Task-specific data.

```typescript
interface AgentDefinition {
  // Routing
  labels: string[];

  // Identity
  taskId: string;
  stageId: string;   // Station ID being executed

  // Prompt (template rendered with task data)
  prompt: string;    // promptTemplate after interpolating {{task.title}}, {{task.description}}
  appendSystemPrompt?: string;

  // Workspace
  repository?: string;   // Git repo to clone (if configured at project level)
  branch?: string;
  initBranch?: boolean;
  autoGit?: boolean;
  commitMessageTemplate?: string;
  devcontainerFiles?: Record<string, string>;

  // Assembly Line Repository
  assemblyLineRepoUrl?: string;   // injected as ASSEMBLY_LINE_REPO_URL
  assemblyLineRepoToken?: string; // injected as ASSEMBLY_LINE_REPO_TOKEN (not logged)

  // Timeouts
  idleTimeoutMinutes?: number;
  maxTimeoutMinutes?: number;
}
```

---

## Label Matching

Labels are how the Server routes Jobs to capable Clients. The rule is:

> **A Runner's registered labels must be a superset of the Station's required labels.**

```
Station requires:  ["linux", "claude-code"]

Runner A labels:   ["linux", "claude-code"]            ✓ matches
Runner B labels:   ["linux", "claude-code", "x86_64"]  ✓ matches (superset)
Runner C labels:   ["linux"]                           ✗ no match
Runner D labels:   ["macos", "claude-code"]            ✗ no match (linux required)
```

When multiple Runners match, the Server dispatches to the first available idle Runner.

---

## Prompt Template Interpolation

The `promptTemplate` field supports these template variables:

| Variable | Source |
|---|---|
| `{{task.title}}` | `TaskData.title` |
| `{{task.description}}` | `TaskData.description` |

The rendered prompt is what the Agent receives. Assembly Line designers use the description field to carry all structured input from the Task Card through to the Agent.

---

## Assembly Line Repository Station Flag

Set `createAssemblyLineRepo: true` on the Station where you want the repository provisioned. The Server creates it before dispatching the Job. All subsequent Stations receive `assemblyLineRepoUrl` and `assemblyLineRepoToken` in their Agent Definition automatically — no need to set `createAssemblyLineRepo` on every Station.

Best practice: enable it on the first Station that writes to the repository (typically INTAKE).
