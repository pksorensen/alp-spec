# 09 — Operator

A **Station Operator** (Operator) is spawned by the Runner for each Job. It manages the entire execution lifecycle at one Station: setting up the workspace, starting the Agent, streaming output to viewers, detecting completion, and exiting with an outcome code.

The Operator is the translation layer between the infrastructure world (Runner, ALP Server) and the AI world (Agent). It knows how to work with a specific type of Agent.

---

## Responsibilities

1. **Receive** the Agent Definition from the Runner (prompt, timeouts, repository config, Assembly Line Repository credentials)
2. **Set up** the execution environment (workspace directory, git clone, git credentials, devcontainer files)
3. **Configure** the Agent's environment (inject `ASSEMBLY_LINE_REPO_URL`, `ASSEMBLY_LINE_REPO_TOKEN`, and other standard env vars)
4. **Start** the Agent with the rendered prompt
5. **Stream** the Agent's terminal output to registered viewers in real time
6. **Detect** when the Agent has finished (exit code, or explicit completion signal)
7. **Exit** with code 0 (success) or non-zero (failure) so the Runner can report the outcome

---

## Operator MCP Server

Every ALP Operator MUST expose an **MCP server** to the Agent process. This is how the Agent signals completion, provides structured output, and accesses any Operator-provided tools.

The Operator starts the MCP server before launching the Agent and passes the server connection details to the Agent via its standard MCP configuration mechanism (e.g. Claude Code's `--mcp-config` flag or the `CLAUDE_MCP_CONFIG` environment variable).

### Required Tool: `complete_station`

Every ALP Operator MCP server MUST expose the `complete_station` tool:

```json
{
  "name": "complete_station",
  "description": "Signal that this station's work is complete. Call this before exiting.",
  "inputSchema": {
    "type": "object",
    "required": ["conclusion"],
    "properties": {
      "conclusion": {
        "type": "string",
        "enum": ["success", "failure"],
        "description": "Whether the station's work was completed successfully"
      },
      "summary": {
        "type": "string",
        "description": "One-sentence summary of what was accomplished or why it failed"
      },
      "exitCode": {
        "type": "integer",
        "default": 0,
        "description": "Numeric exit code (0 = success, non-zero = failure)"
      }
    }
  }
}
```

When the Agent calls `complete_station`, the Operator:
1. Records the conclusion and summary
2. Prepares to exit with the appropriate exit code
3. Streams a final status to any connected viewers
4. Exits — the Runner reads the exit code and reports the Job outcome to the Server

### Additional Tools (Optional)

The Operator MAY expose additional MCP tools beyond `complete_station`. The reference implementation (vibecast) exposes tools for output streaming and task metadata access. Operators are encouraged to expose tools that are useful for their Agent type.

### Fallback: Stop Hook

The Operator MUST also register a stop hook (or equivalent session-end handler) so that `complete_station` is called automatically if the Agent exits without calling it explicitly. This ensures clean completion even on crashes, timeouts, or force-exits.

For Claude Code, this is a `stop` hook in the Claude Code hooks configuration. The hook reads the session's exit state and calls `complete_station` with the inferred conclusion.

**Result:** The Operator always exits cleanly, regardless of whether the Agent was well-behaved.

---

## Completion Signaling (Summary)

```
Agent ──(calls MCP tool)──► complete_station(conclusion: "success", summary: "...")
                                      │
                              Operator records outcome
                                      │
                              Operator exits with code 0
                                      │
                Runner ──(reads exit code)──► reports { jobResult: "success" } to Server
```

For the crash/timeout path:
```
Agent exits unexpectedly (crash / timeout)
        │
Stop hook fires ──► complete_station(conclusion: "failure", summary: "session ended unexpectedly")
        │
Operator exits with non-zero code
        │
Runner ──► reports { jobResult: "failed" } to Server
```

---

## Standard Environment Variables

The Operator MUST inject these into the Agent's environment:

| Variable | Description |
|---|---|
| `ASSEMBLY_LINE_REPO_URL` | Clone URL for this Task's Assembly Line Repository |
| `ASSEMBLY_LINE_REPO_TOKEN` | Auth token for the Assembly Line Repository (never logged) |
| `AGENTICS_JOB_ID` | The Job ID being executed |
| `AGENTICS_TOKEN` | Server auth token for Agent-to-Server callbacks |
| `AGENTICS_BASE_URL` | Server base URL |
| `AGENTICS_OWNER` | Project owner |
| `AGENTICS_PROJECT_NAME` | Project name |

---

## Output Streaming

The Operator SHOULD stream the Agent's terminal output in real time. This enables live observation of the Agent's work — users can watch the Agent think, code, and commit in real time via the Server's viewer UI.

The reference implementation (vibecast) uses:
- **tmux** — manages the Agent's terminal session, handles resize
- **ttyd** — serves the tmux session over WebSocket
- **ws-relay** — the ALP Server's WebSocket relay that fans out the stream to multiple viewers

See [docs/terminal-sizing.md](../../docs/terminal-sizing.md) for terminal dimension details.

---

## Sandbox Context

The Operator runs **inside the devcontainer sandbox** created by the Runner. This means:
- It has no access to the host filesystem except explicit bind mounts
- It cannot talk to the Docker daemon directly
- It has access to `/run/alp/cred.sock` (mounted by the Runner) for credential requests
- All outbound HTTP/HTTPS routes through the Runner's Egress Proxy (`HTTP_PROXY` env var)

For git credential injection, the Operator SHOULD call the Credential Server to retrieve a scoped token rather than relying on env-var secrets. This is consistent with the broader ALP security model where the Agent never holds real credentials.

> For the full security model, see [11-security.md](11-security.md).

---

## Workspace Setup

Before starting the Agent, the Operator SHOULD:

1. Create an isolated working directory for this Job (e.g. `/tmp/vibecast-job-{jobId}`)
2. Clone the project repository (if `agentDefinition.repository` is set)
3. Check out the correct branch (or create one if `initBranch: true`)
4. Configure git credentials for the Assembly Line Repository:
   ```bash
   git config credential.helper store
   echo "https://git:{ASSEMBLY_LINE_REPO_TOKEN}@{host}" >> ~/.git-credentials
   ```
5. Write any `devcontainerFiles` to their paths
6. Write the prompt to a file (e.g. `initial-prompt.txt`)

---

## Reference Implementation

**vibecast** — open source Go CLI. Uses tmux + ttyd + Claude Code.

- Starts Claude Code with: `claude --prompt-file initial-prompt.txt`
- Exposes `stop_broadcast` as an MCP tool to Claude Code
- Registers a Claude Code stop hook for graceful fallback completion
- Streams terminal output via ttyd → ws-relay → browser viewer
- Connects to agentics.dk WebSocket relay for live stream distribution

---

## Implementing Your Own Operator

An ALP Operator MUST:
1. Accept an Agent Definition payload
2. Start the Agent with the prompt
3. Exit with 0 on success, non-zero on failure

An ALP Operator SHOULD:
1. Inject the standard environment variables
2. Respect `idleTimeoutMinutes` and `maxTimeoutMinutes`
3. Stream output for real-time observability
4. Set up git credentials for the Assembly Line Repository

An ALP Operator MAY:
1. Expose a structured completion tool to the Agent (e.g. the `stop_broadcast` MCP tool pattern)
2. Run the Agent inside a devcontainer for isolation
3. Support multiple Agent types based on the Station's labels
