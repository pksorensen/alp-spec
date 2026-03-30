# 01 — Overview

**Assembly Line Protocol (ALP)** is an open protocol for running AI agent pipelines at scale. It defines how work flows through a sequence of AI-powered Stations, and the contracts between the four roles that make it happen: Server, Runner, Operator, and Agent.

---

## The Problem ALP Solves

Building a multi-step AI pipeline today requires stitching together ad-hoc scripts, custom queues, and one-off integrations. Every team rebuilds the same plumbing: how does work get dispatched to an AI agent? How does the agent signal it's done? How do you chain multiple agents sequentially? How do you handle failures, retries, and human review gates?

ALP standardises this plumbing the same way MCP standardised tool calling, and the same way GitHub Actions standardised CI/CD workflow execution. The protocol is simple enough to implement in an afternoon, but powerful enough to run production AI pipelines.

---

## Architecture

```
╔══════════════════════════════════════════════════════════════╗
║  ALP SERVER  (e.g. agentics.dk)                              ║
║                                                              ║
║  ┌─────────────────┐  ┌───────────────┐  ┌───────────────┐  ║
║  │ Assembly Lines  │  │  Task Queue   │  │Runner Registry│  ║
║  │ Stations        │  │  Jobs         │  │  Labels       │  ║
║  │ Transition Rules│  │  Gates        │  │  Tokens       │  ║
║  └─────────────────┘  └───────────────┘  └───────────────┘  ║
╚══════════════════════════╤═══════════════════════════════════╝
                           │  HTTP (short-poll + Bearer token)
                           │
╔══════════════════════════╧═══════════════════════════════════╗
║  CLIENT / RUNNER  (e.g. pks-cli)                             ║
║                                                              ║
║  Polls for jobs · Claims tasks · Reports outcomes            ║
╚══════════════════════════╤═══════════════════════════════════╝
                           │  spawns per-job
                           │
╔══════════════════════════╧═══════════════════════════════════╗
║  OPERATOR  (e.g. vibecast)                                   ║
║                                                              ║
║  Sets up workspace · Runs Agent · Streams output             ║
║  Detects completion · Maps exit code to outcome              ║
╚══════════════════════════╤═══════════════════════════════════╝
                           │  exec
                           │
╔══════════════════════════╧═══════════════════════════════════╗
║  AGENT  (e.g. Claude Code)                                   ║
║                                                              ║
║  Reads prompt · Does the work · Exits with result            ║
║  Reads/writes Assembly Line Repository (shared workspace)    ║
╚══════════════════════════════════════════════════════════════╝
```

---

## The Four Roles

### Server
The Server is the authoritative source of truth. It defines Assembly Lines (the pipeline templates), manages the Task queue (work in flight), and maintains the Runner registry. When a Task reaches a Station, the Server dispatches a Job to a matching Runner.

The Server does not execute any AI work itself — all execution happens on Runners via Operators.

→ See [07-server.md](07-server.md)

### Runner
The Runner is a long-running daemon that registers with the Server and polls for Jobs. When a Job arrives, the Runner spawns a Station Operator to handle execution. When the Operator finishes, the Runner reports the outcome back to the Server.

**The Runner is not the Agent.** It is the infrastructure layer. A single Runner can manage multiple Operators running different Agent types simultaneously.

→ See [08-runner.md](08-runner.md)

### Operator (Station Operator)
The Operator is spawned by the Runner for each Job. It sets up the execution environment, starts the Agent, monitors for completion, and streams output to viewers. The Operator knows how to work with a specific type of Agent — it is the translation layer between the infrastructure world (Runner, Server) and the AI world (Agent).

→ See [09-operator.md](09-operator.md)

### Agent
The Agent is the AI that executes the work at a Station. It receives a prompt and context from the Operator and produces output. The Agent has no direct knowledge of the Server, the Assembly Line, or other Stations. It reads and writes to the Assembly Line Repository when it needs to share state with other Stations.

→ See [10-agent.md](10-agent.md)

---

## A Task's Journey

To make this concrete, here is a minimal example: a two-station Assembly Line that writes a poem and then reviews it.

1. A user submits a Task Card: `title: "Poem about robots"`, `description: "Write something creative for our homepage."`
2. The Server places the Task in the queue at Station 1 (WRITE).
3. A Runner polls and receives the Job. It spawns an Operator.
4. The Operator starts Claude Code with the WRITE prompt: *"Write a short poem about robots. Exit when done."*
5. Claude writes the poem, exits 0.
6. The Operator signals completion to the Runner (exit code 0).
7. The Runner reports `{ jobResult: "success" }` to the Server.
8. The Server applies the default Transition Rule (success → advance) and moves the Task to Station 2 (REVIEW).
9. Steps 3–7 repeat for REVIEW.
10. After REVIEW succeeds, the Server marks the Task `completed`.

---

## Relationship to MCP

MCP defines a protocol for AI models to call tools. ALP defines a protocol for AI agents to be dispatched as workers in a pipeline. They are complementary:

- An ALP Operator might use MCP to expose tools to the Agent (Claude Code uses MCP extensively)
- An ALP Agent might itself be an MCP client that calls external services
- An ALP Server could expose its job dispatch as an MCP tool (submit a task from within an agent)

---

## Relationship to GitHub Actions

| GitHub Actions | ALP |
|---|---|
| Workflow | Assembly Line |
| Job | Station |
| Job run | Job (dispatched to a Runner) |
| Self-hosted runner | Runner |
| Runner agent process | Operator |
| The code that runs | Agent |
| `runs-on: [linux, self-hosted]` | Station labels |
| `if: success()` | Transition Rule condition |

The key difference: GitHub Actions jobs run shell commands. ALP Stations run AI agents. The polling protocol, label-based routing, and job result reporting are all directly inspired by GitHub's self-hosted runner protocol.
