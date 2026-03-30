# 02 — Concepts & Glossary

A reference for every term used in the ALP specification. If you encounter an unfamiliar term in another document, look it up here.

---

## Core Terms

### Assembly Line Protocol (ALP)
The formal name of this specification. ALP defines the contracts between the Server, Client, Operator, and Agent roles. A server that implements the ALP API is an **ALP-compliant server**. A client that speaks the ALP polling protocol is an **ALP client**.

### Assembly Line
An ordered sequence of Stations that defines how a type of work is processed. An Assembly Line is a template — it does not run by itself. When a Task is submitted against an Assembly Line, a Task instance is created and flows through the Stations one by one.

*Analogous to: GitHub Actions workflow, factory production line*

### Station
One discrete unit of work in an Assembly Line. Each Station has an Agent Definition that specifies what prompt and configuration to give the Agent. When a Task reaches a Station, the Server dispatches a Job to a matching Client.

*Analogous to: GitHub Actions job, factory workstation*

### Station Operator — see Operator

### Task
The unit of work that flows through an Assembly Line from start to finish. A Task is created when a user (or external trigger) submits a Task Card. One Assembly Line can have many Tasks in flight simultaneously, each at a different Station.

*Analogous to: GitHub Actions workflow run, factory work order*

### Task Card
The submission form that creates a Task. A Task Card has two required fields: `title` (a short name) and `description` (the full context, in markdown). Everything the Agent needs to understand the work should be in the description.

### Job
A specific Station execution dispatched to a Client. When a Task reaches a Station, the Server creates a Job and queues it. One Task produces one Job per Station it reaches. If a Station is retried, a new Job is created.

### Run
A single execution of a Job on a specific Client. One Job has one Run in the normal case. A retry creates a new Run. (Note: in the current implementation, Job and Run are often used interchangeably at the API level.)

### Transition Rule
A condition that controls when and where a Task moves after a Station completes. Without Transition Rules, the default behavior applies (success → next Station, failure → halt). With Transition Rules, you can express human review gates, retries, and (in a future version) conditional branching.

*Analogous to: GitHub Actions `if: success()` condition, factory routing rule*

### Gate (Human Review Gate)
A pause point in an Assembly Line. When a Task reaches a Gate, it enters the `awaiting_review` state and waits until a human approves or rejects it. Defined as part of a Transition Rule.

### Assembly Line Repository
A hosted git repository provisioned per Task, used as a shared workspace for Agents across all Stations. Each Station's Agent clones it, writes its outputs (reports, generated files, artifacts), commits, and pushes. The next Station clones it and reads the accumulated outputs.

The Assembly Line Repository is the primary mechanism for passing state between Stations. It also serves as a full audit trail — every file produced by every Agent is versioned in git.

*Analogous to: GitHub Actions artifact storage, factory shared workbench*

### Server
The ALP service that manages Assembly Lines, the Task queue, and the Client registry. The Server is the authoritative coordinator — it decides which Client gets which Job based on label matching, and it applies Transition Rules when Jobs complete.

*Reference implementation: agentics.dk (closed source)*

### Client
A long-running process that registers with an ALP Server, polls for Jobs, and delegates execution to an Operator. The Client is infrastructure — it has no AI capability of its own. It is the glue between the Server and the Operator/Agent stack.

Also called **Runner**. A single Client can proxy to multiple Operators running different Agent types.

*Reference implementation: pks-cli (open source, C#)*

### Operator (Station Operator)
The component spawned by the Client for each Job. It prepares the execution environment (workspace, git credentials, environment variables), starts the Agent, streams output to viewers, and detects completion. The Operator knows how to run a specific type of Agent — it is the translation layer between infrastructure and AI.

*Reference implementation: vibecast (open source, Go) — uses tmux + ttyd + Claude Code*

### Agent
The AI that executes the Station's task. It receives a prompt and context from the Operator and produces output by working in the execution environment. The Agent has no direct knowledge of the Server, the Assembly Line, or ALP itself. From the Agent's perspective, it is given a task to complete and an environment to work in.

*Reference implementation: Claude Code*

### Agent Definition
The payload the Server sends to the Client when dispatching a Job. It contains everything the Client and Operator need to configure and run the Agent: the prompt, repository to clone, labels, timeouts, and the Assembly Line Repository credentials.

### Label
A capability tag on a Station or a Client. The Server matches Station labels to Client labels when dispatching Jobs. A Client's registered labels must be a superset of the Station's required labels for a match.

Examples: `linux`, `claude-code`, `gpu`, `x86_64`, `vibecheck`

---

## The Critical Distinction: Client vs Agent

> **The Client is not the Agent.**

| | Client | Agent |
|---|---|---|
| **What it is** | Infrastructure daemon | AI model |
| **Knows about** | ALP Server, polling, tokens | Prompt, workspace, tools |
| **Communicates with** | ALP Server (HTTP) | Operator (process I/O, MCP) |
| **Lifetime** | Long-running, always on | Per-job, exits when done |
| **Examples** | pks-cli | Claude Code, GPT-4 CLI |

A Client can manage multiple Operators simultaneously, each running a different type of Agent. The Client does not decide *what* the Agent does — that is determined by the Station's Agent Definition on the Server.
