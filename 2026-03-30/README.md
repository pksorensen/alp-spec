# Assembly Line Protocol (ALP)

> A protocol for running AI agent pipelines at scale — inspired by GitHub Actions self-hosted runners and the Model Context Protocol (MCP).

---

## What Is ALP?

The **Assembly Line Protocol (ALP)** is an open protocol that describes how AI tasks flow through a sequence of automated Stations, each powered by an AI Agent. It defines the contracts between four roles so that anyone can build a compatible Server or Client:

- **Server** — manages Assembly Lines, the task queue, and the Client registry
- **Client** (alias: Runner) — polls the Server for jobs, delegates execution to a Station Operator, and reports results back
- **Operator** — spawned by the Client to manage a single Station's execution (sets up the workspace, runs the Agent, streams output)
- **Agent** — the AI that does the actual work at each Station (e.g. Claude Code)

The key inspiration is **GitHub self-hosted runners**: just as any machine can register as a runner and execute GitHub Actions workflows, any ALP Client can register with a compliant ALP Server and execute Stations. You can run your own Clients against the agentics.dk Server, or build your own ALP Server that pks-cli can connect to.

---

## Why ALP Exists

Today, agentics.dk runs Assembly Lines internally. ALP exists so that:

1. **Anyone can build a Client** — connect your own runner/operator/agent stack to the agentics.dk server or any other ALP-compliant server
2. **Anyone can build a Server** — implement the ALP server API and route tasks to any Client that speaks the protocol
3. **The ecosystem grows** — just as MCP unlocked a marketplace of AI tools, ALP should unlock a marketplace of AI production pipelines

The reference implementations are:
- **Server**: [agentics.dk](https://agentics.dk) — closed source
- **Client / Runner**: [pks-cli](https://github.com/pksorensen/pks-cli) — open source, C#
- **Operator**: [vibecast](https://github.com/pksorensen/agentic-live) — open source, Go

---

## Who Should Read This

| You want to... | Read... |
|---|---|
| Understand the big picture | [spec/01-overview.md](spec/01-overview.md) |
| Learn the terminology | [spec/02-concepts.md](spec/02-concepts.md) |
| Understand Assembly Lines | [spec/03-assembly-lines.md](spec/03-assembly-lines.md) |
| Understand Stations | [spec/04-stations.md](spec/04-stations.md) |
| Understand Tasks | [spec/05-tasks.md](spec/05-tasks.md) |
| Understand flow control | [spec/06-transition-rules.md](spec/06-transition-rules.md) |
| Build a **Server** | [spec/07-server.md](spec/07-server.md) |
| Build a **Client/Runner** | [spec/08-client.md](spec/08-client.md) |
| Build an **Operator** | [spec/09-operator.md](spec/09-operator.md) |
| Understand Agents | [spec/10-agent.md](spec/10-agent.md) |
| See a real-world example | [examples/vibecheck.md](examples/vibecheck.md) |
| Start minimal | [examples/hello-world.md](examples/hello-world.md) |

---

## Spec Status

This specification is in **active draft**. Open questions are tracked in [todo.md](todo.md). Sections with remaining open questions are marked `> ❓`.
