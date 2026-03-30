# 05 — Tasks

A **Task** is the unit of work that flows through an Assembly Line. It is created when a user (or an external trigger) submits a Task Card. The Task then progresses through each Station until it is `completed` or `failed`.

---

## Task Card

A **Task Card** is what a user fills in to create a Task. It has two required fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `title` | string | yes | Short name for the work (e.g. `"Vibecheck: my-org/backend"`) |
| `description` | string | yes | Full context for the Agent, in markdown |

The `description` field carries all the structured inputs the Agent needs. Assembly Line designers document the expected description format for their pipeline. For example, the Vibecheck Assembly Line expects the description to contain a repository URL, branch, and optionally a stack description and areas of concern.

There are no typed input fields in v1. The Agent reads the description and acts on its content.

**Optional task fields:**

| Field | Type | Description |
|---|---|---|
| `priority` | `"low"` \| `"medium"` \| `"high"` | Scheduling priority (default: `"medium"`) |
| `labels` | string[] | Custom labels for tracking (not used for routing — Station labels handle routing) |

---

## Task Lifecycle

A Task moves through the following states as it flows through an Assembly Line:

```
[Submit Task Card]
        │
        ▼
     queued          ← waiting for a Runner to claim the Job at the current Station
        │
        ▼
     claimed         ← a Runner has taken the Job and is starting the Operator
        │
        ▼
    in_progress      ← the Operator has started the Agent, work is underway
        │
   ┌────┴────┐
   │         │
   ▼         ▼
success    failure
   │         │
   │         └──────────────────────► failed  (pipeline stops)
   │
   ├── [not last Station] ──────────► queued  (at next Station)
   │
   ├── [last Station] ──────────────► completed
   │
   └── [Transition Rule: gate] ────► awaiting_review
                                          │
                                    ┌─────┴─────┐
                                    │           │
                                  approve     reject
                                    │           │
                                    ▼           ▼
                                 queued       failed
                              (next Station)
```

### State Descriptions

| State | Description |
|---|---|
| `queued` | Waiting for a matching Runner to poll and claim the Job |
| `claimed` | A Runner has claimed the Job; Operator is starting |
| `in_progress` | Operator has started the Agent; work is underway |
| `awaiting_review` | Task has reached a Human Review Gate and is paused pending approval |
| `completed` | All Stations finished successfully |
| `failed` | A Station failed with no applicable retry or recovery rule |

---

## Task Data

The full Task object maintained by the Server:

```typescript
interface TaskData {
  id: string;
  owner: string;
  projectId: string;
  stageId: string;            // Assembly Line ID
  title: string;
  description?: string;
  columnId: string;           // Current Station ID
  order: number;              // Position within the Station's queue
  priority?: 'low' | 'medium' | 'high';
  labels?: string[];
  pendingGateId?: string;     // Set when task is in awaiting_review state
  assemblyLineRepo?: AssemblyLineRepo;  // Provisioned when createAssemblyLineRepo fires
  createdAt: number;          // Unix ms
  updatedAt: number;          // Unix ms
}

interface AssemblyLineRepo {
  repoId: string;
  url: string;    // https://server/api/git/owner/project/repoId.git
  branch: string; // main
  createdAt: number;
}
```

---

## Task Submission API

```
POST /api/owners/{owner}/projects/{project}/stages/{assemblyLineId}/tasks
Authorization: Bearer {token}

{
  "title": "Vibecheck: my-org/backend",
  "description": "Repository: https://github.com/my-org/backend\nBranch: main\nConcerns: auth implementation",
  "columnId": "intake",
  "priority": "high"
}
```

**Response:** The created `TaskData` object.

If the first Station is configured with `type: "dispatch_worker"`, the Server automatically creates a Job and queues it for dispatch after creating the Task.

---

## Per-Station Progress

A Task at Station 3 of 6 shows progress via `columnId` (the current Station ID). The Server maintains the history of which Stations the Task has already passed through, enabling the UI to display a progress indicator.
