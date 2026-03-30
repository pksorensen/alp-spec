# 07 — Server

The **ALP Server** is the authoritative coordinator of an Assembly Line deployment. It stores Assembly Line definitions, manages the Task queue, tracks the Runner registry, dispatches Jobs, and applies Transition Rules when Jobs complete.

The Server does not execute any AI work. All execution happens on Clients via their Operators.

---

## Responsibilities

- Store and serve Assembly Line definitions (CRUD)
- Accept Task submissions and manage Task lifecycle
- Maintain the Runner registry (registrations + labels)
- Match Station labels to available Clients when dispatching Jobs
- Dispatch Jobs to matching idle Clients
- Receive Job results and apply Transition Rules
- Provision and manage Assembly Line Repositories
- Enforce Gate approvals

---

## Required API Surface

A compliant ALP Server MUST implement the following endpoints. URL paths are the reference implementation paths — other implementations may use different URL structures but must support the same semantics.

---

### Runner Registration

```
POST /api/owners/{owner}/projects/{project}/runners/register
Authorization: Bearer {userToken}

{
  "name": "my-runner-1",
  "labels": ["linux", "claude-code", "x86_64"]
}
```

Registers a new Runner with the Server. Returns a Runner ID and a long-lived token for all subsequent requests.

**Response:**
```json
{
  "id": "runner-abc123",
  "token": "eyJ...",
  "registeredAt": "2026-03-30T12:00:00Z"
}
```

---

### Job Polling

```
POST /api/owners/{owner}/projects/{project}/runners/jobs
Authorization: Bearer {runnerToken}
```

The Runner calls this to check for available Jobs. The Server returns the highest-priority Job whose Station labels are a subset of the Runner's registered labels.

**Response — no jobs available:**
```
HTTP 204 No Content
```

**Response — job available:**
```
HTTP 200 OK

{
  "jobs": [
    {
      "id": "job-xyz789",
      "runId": "run-001",
      "agentDefinition": {
        "prompt": "You are a security expert...",
        "labels": ["linux", "claude-code"],
        "taskId": "task-abc123",
        "stageId": "security",
        "idleTimeoutMinutes": 30,
        "maxTimeoutMinutes": 60,
        "assemblyLineRepoUrl": "https://agentics.dk/api/git/owner/project/repo123.git",
        "assemblyLineRepoToken": "ghs_..."
      }
    }
  ]
}
```

---

### Job Result Reporting

```
PATCH /api/owners/{owner}/projects/{project}/runners/{runnerId}
Authorization: Bearer {runnerToken}

{
  "jobResult": "success",
  "exitCode": 0,
  "error": null
}
```

The Runner calls this when a Job completes (success or failure). The Server applies the relevant Transition Rules and advances the Task.

**`jobResult` values:** `"success"` | `"failed"` | `"in_progress"` (for intermediate heartbeats)

**Response:** `HTTP 200 OK`

---

### Gate Approval

```
POST /api/owners/{owner}/projects/{project}/stages/{assemblyLineId}/tasks/{taskId}/gates/{gateId}
Authorization: Bearer {userToken}

{
  "action": "approve",
  "reason": "Looks good, ship it"
}
```

Approves or rejects a Task at a Human Review Gate.

**`action` values:** `"approve"` | `"reject"`

---

## Authentication

> **⚠️ Work in progress.** Authentication in ALP is an area of active development. The current implementation uses Bearer tokens. The details below reflect the current state and will be formalised in a future version of the spec.

All ALP API requests MUST include:
```
Authorization: Bearer {token}
```

Two token types exist:
- **User token** — authenticates a human user for management operations (submit tasks, approve gates, register Clients). Issued via the Server's own auth mechanism (agentics.dk uses OAuth).
- **Runner token** — authenticates a Runner for polling and result reporting. Issued at Runner registration time and stored by the Runner.

How user tokens are issued is the Server's implementation detail. A Server MAY use OAuth 2.0 device flow, API keys, or any other mechanism.

---

## Optional: Server-Sent Events (SSE)

A Server MAY additionally expose an SSE endpoint to push job-available notifications to Clients, eliminating polling latency:

```
GET /api/owners/{owner}/projects/{project}/runners/events
Authorization: Bearer {runnerToken}

# Server pushes:
event: job_available
data: {"jobId": "job-xyz789"}
```

When a Runner receives a `job_available` event, it should immediately call the poll endpoint to claim the Job. SSE is an optimisation — all ALP Servers must support short-polling as the baseline.

---

## Reference Implementation

**agentics.dk** — closed-source, hosted reference implementation. See [https://agentics.dk](https://agentics.dk).
