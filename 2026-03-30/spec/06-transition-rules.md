# 06 — Transition Rules

**Transition Rules** define when and where a Task moves after a Station completes. Without Transition Rules, the Assembly Line uses its default behavior (success → advance, failure → halt). Transition Rules let you override this: require human review before certain Stations, configure retries, or (in a future version) route conditionally based on the Agent's output.

---

## Default Behavior (No Rules)

When no Transition Rules are defined, the Server applies these defaults:

| Outcome | Action |
|---|---|
| Station succeeds + not last Station | Advance Task to next Station |
| Station succeeds + last Station | Task → `completed` |
| Station fails | Task → `failed`, pipeline stops |

---

## Transition Rule Structure (v1)

```typescript
interface TransitionRule {
  id: string;
  fromStationId: string;        // Which Station this rule applies to
  toStationId: string;          // Where to send the Task (overrides default "next")
  condition: 'success' | 'failure' | 'cancelled' | 'any';
  gateId?: string;              // If set, Task pauses here for human review before advancing
}
```

A Transition Rule fires when:
1. The Job for `fromStationId` completes
2. The Job's outcome matches `condition`
3. The Task advances to `toStationId` (possibly via a Gate)

Multiple rules can apply to the same Station with different conditions (e.g. one for success, one for failure).

---

## Example Rules

### Human Review Gate

Require a human to approve before the DELIVER Station runs:

```json
{
  "id": "synthesis-to-deliver",
  "fromStationId": "synthesis",
  "toStationId": "deliver",
  "condition": "success",
  "gateId": "human-review"
}
```

### Explicit Retry

Route back to the same Station on failure (max retries enforced by the Gate or a counter):

```json
{
  "id": "security-retry",
  "fromStationId": "security",
  "toStationId": "security",
  "condition": "failure"
}
```

### Skip to a Different Station

On failure in ARCHITECTURE, jump straight to SYNTHESIS (skip intermediate stations):

```json
{
  "id": "arch-skip",
  "fromStationId": "architecture",
  "toStationId": "synthesis",
  "condition": "failure"
}
```

---

## Gates (Human Review)

A Gate is a named pause point. When a Task reaches a Transition Rule with a `gateId`, it enters the `awaiting_review` state and waits for a human to act.

```typescript
interface GateData {
  id: string;
  name: string;
  description?: string;
  requiresApproval: boolean;
  notifyEmails?: string[];
}
```

**On approve** → Task advances to the `toStationId` in the Transition Rule.
**On reject** → Task is marked `failed` with a rejection reason.

The Server exposes an API endpoint for approving/rejecting Gates:

```
POST /api/owners/{owner}/projects/{project}/stages/{assemblyLineId}/tasks/{taskId}/gates/{gateId}
Authorization: Bearer {token}

{ "action": "approve" | "reject", "reason": "Looks good / reason for rejection" }
```

---

## Condition Semantics

| Condition | Fires when... |
|---|---|
| `success` | Job completed with `jobResult: "success"` |
| `failure` | Job completed with `jobResult: "failed"` |
| `cancelled` | Job was cancelled (timeout or manual cancellation) |
| `any` | Job completed for any reason (success, failure, or cancelled) |

The Agent signals its outcome via **exit code**: exit 0 = success, non-zero = failure. The Operator maps this to `jobResult` and reports it to the Server. The Agent can express nuanced failure modes by choosing different non-zero exit codes, but Transition Rules in v1 see only the binary success/failure.

---

## Future Directions: Agentic Conditions

> This section describes planned v2 capabilities. Nothing below is part of the current specification.

The v1 condition types are outcome-based (deterministic). A planned extension adds two new condition types that allow smarter routing based on *what the Agent produced*, not just *whether it succeeded*.

### Agentic Condition

An LLM evaluates a yes/no question against the Station's output. The answer determines which Transition Rule fires:

```json
{
  "id": "security-escalate",
  "fromStationId": "security",
  "condition": "agentic",
  "evaluator": "Did the agent find any HIGH or CRITICAL severity security issues?",
  "onYes": { "toStationId": "escalate" },
  "onNo":  { "toStationId": "synthesis" }
}
```

The evaluator is run by a designated evaluator Client (or the Server itself). It reads the Assembly Line Repository and answers the question based on the Agent's outputs.

### Hook Condition

A script or tool in the Assembly Line Repository is called. Exit 0 = pass (advance), non-zero = block:

```json
{
  "id": "security-hook",
  "fromStationId": "security",
  "condition": "hook",
  "hook": ".agentics/hooks/security-gate.sh",
  "onPass":  { "toStationId": "synthesis" },
  "onBlock": { "toStationId": "security", "feedbackFromHook": true }
}
```

### Block and Feedback

When an agentic or hook condition **blocks** a transition, the Task is not simply marked failed — it is sent back to the current Station with the evaluator's feedback appended to the prompt. This enables self-correcting pipelines where the Agent can retry with guidance:

```
First run:   Agent receives original prompt
             → Agent exits 0 → agentic evaluator says "block" (found HIGH issues)

Second run:  Agent receives original prompt +
             "FEEDBACK FROM PREVIOUS EVALUATION:
              The security report found 3 HIGH severity issues in auth.ts.
              Please address these specifically before completing."
```

This pattern is directly inspired by Claude Code stop hooks, which can block a session and provide corrective feedback to the Agent.
