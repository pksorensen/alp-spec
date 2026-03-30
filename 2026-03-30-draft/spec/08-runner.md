# 08 — Runner

An **ALP Runner** is a long-running daemon that registers with an ALP Server, polls for Jobs, delegates execution to a Station Operator, and reports results back to the Server.

> **The Runner is not the Agent.** The Runner is infrastructure. The AI capability lives in the Agent, which is managed by the Operator.

---

## Responsibilities

1. Register with the Server and persist the Runner ID and token
2. Poll the Server for available Jobs (short-poll or SSE)
3. Claim a Job and spawn a Station Operator
4. Monitor the Operator until it completes
5. Report the Job outcome to the Server

---

## Registration

Before polling, a Runner must register with the Server.

**Required at registration:**
- `name` — human-readable identifier for this Runner instance
- `labels` — capability declarations (e.g. `["linux", "claude-code", "x86_64"]`)

**Stored after registration:**
- `id` — Runner ID
- `token` — Bearer token for all subsequent authenticated requests
- `server`, `owner`, `project` — the Server endpoint and scope

Registration is persistent — the Runner stores its credentials locally (e.g. `~/.pks-cli/agentics-runners.json`) and reuses them across restarts.

---

## Polling Loop

The Runner runs a continuous polling loop:

```
loop:
  response = POST /runners/jobs  (Authorization: Bearer {token})

  if response.status == 204:
    sleep(pollingInterval)   // default: 10 seconds
    continue

  if response.status == 200:
    job = response.body.jobs[0]
    outcome = executeJob(job)
    PATCH /runners/{id}  { jobResult: outcome.result, exitCode: outcome.exitCode, error: outcome.error }
```

**Default polling interval:** 10 seconds. Configurable via `--polling-interval` flag in pks-cli.

If the Server supports SSE, the Runner MAY subscribe to `GET /runners/events` and call the poll endpoint immediately on receiving a `job_available` event, reducing latency from up to 10s to near-zero.

---

## Operator Spawning

When a Job is received, the Runner spawns a Station Operator process with the full Agent Definition as context. The Operator is given:

- The rendered prompt (from the Agent Definition)
- Environment variables: `ASSEMBLY_LINE_REPO_URL`, `ASSEMBLY_LINE_REPO_TOKEN`, `AGENTICS_JOB_ID`, `AGENTICS_TOKEN`, `AGENTICS_BASE_URL`, `AGENTICS_OWNER`, `AGENTICS_PROJECT_NAME`
- A working directory for this Job
- Any `devcontainerFiles` from the Agent Definition

The Runner monitors the Operator process. When the Operator exits, the Runner reads the exit code and reports the outcome.

---

## Job Outcome Schema

When a Job completes, the Runner MUST report the following to the Server:

```typescript
interface JobOutcome {
  jobResult: 'success' | 'failed';
  exitCode: number;
  error: string | null;
}
```

The Runner SHOULD also send intermediate `jobResult: "in_progress"` updates (heartbeats) for long-running Jobs to indicate the Job is still alive.

**How the exit code is determined:** The Operator process exits with 0 for success and non-zero for failure. The Runner maps the Operator's exit code to `jobResult`. The details of how the Operator elicits a structured outcome from the Agent are the Operator's implementation concern — see [09-operator.md](09-operator.md).

---

## Label Matching and Capability Declaration

A Runner's labels are what the Server uses to route Jobs to it. Labels should describe:

| Category | Examples |
|---|---|
| OS | `linux`, `macos`, `windows` |
| Agent type | `claude-code`, `gpt4-cli` |
| Hardware | `gpu`, `x86_64`, `arm64` |
| Custom capability | `vibecheck`, `high-memory` |

The Station's `labels` array lists what is **required**. The Runner's registered labels must be a **superset** of the Station's requirements.

---

## Concurrency

A single Runner instance handles one Job at a time (per pks-cli's current implementation). For concurrency, run multiple Runner instances — each registers with a unique name and the Server dispatches Jobs across them.

---

## Implementing Your Own ALP Runner

A compliant ALP Runner MUST:

1. Register with `POST /runners/register` and persist the token
2. Poll `POST /runners/jobs` in a loop with `Authorization: Bearer {token}`
3. On `HTTP 200`: execute the Job (via an Operator) and report the result
4. On `HTTP 204`: wait for the polling interval, then retry
5. Report result via `PATCH /runners/{id}` with the Job Outcome schema

A compliant ALP Runner SHOULD:

- Respect `idleTimeoutMinutes` and `maxTimeoutMinutes` from the Agent Definition
- Send `in_progress` heartbeats for long-running Jobs
- Handle SIGTERM gracefully (complete current Job, then stop)
- Support SSE for reduced polling latency when the Server offers it

---

## Reference Implementation

**pks-cli** — open source C# CLI.

Install:
```bash
# TODO: add installation instructions when publicly released
```

Start a runner:
```bash
pks agentics runner start \
  --server https://agentics.dk \
  --owner pksorensen \
  --project my-project \
  --labels linux,claude-code \
  --polling-interval 10
```
