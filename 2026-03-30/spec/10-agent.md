# 10 — Agent

The **Agent** is the AI that executes the work at a Station. It is the intelligence in the Assembly Line — everything else (Server, Client, Operator) is plumbing to get the right context to the Agent and collect its output.

---

## What the Agent Knows (and Doesn't)

The Agent has no direct knowledge of:
- The ALP Server
- The Assembly Line or which Station it is at
- Other Stations' outputs (except what it finds in the Assembly Line Repository)
- The Client that dispatched the Job

From the Agent's perspective, it is given a task to complete and an environment to work in. The Agent simply executes the task described in its prompt.

---

## What the Agent Receives

The Operator gives the Agent:

### Prompt
The primary instruction — written to a file (e.g. `initial-prompt.txt`) and passed to the Agent at startup. The prompt comes from the Station's `promptTemplate` after interpolating `{{task.title}}` and `{{task.description}}`. It tells the Agent what to do at this Station.

### System Prompt
An optional `appendSystemPrompt` from the Station trigger is appended to the Agent's system prompt. This is typically used to inject pipeline-wide behavioral instructions (e.g. "always commit and push your outputs before exiting").

### Environment Variables

| Variable | Description |
|---|---|
| `ASSEMBLY_LINE_REPO_URL` | Clone URL for this Task's Assembly Line Repository |
| `ASSEMBLY_LINE_REPO_TOKEN` | Auth token for the repo (pre-configured in git credentials by the Operator) |
| `AGENTICS_JOB_ID` | The Job ID (useful for logging) |
| `AGENTICS_BASE_URL` | Server base URL |

### Workspace
A directory containing:
- The cloned project repository (if `agentDefinition.repository` is set)
- The initial prompt file
- Any `devcontainerFiles` injected by the Operator

---

## Assembly Line Repository

The Assembly Line Repository is the primary way Agents share state across Stations. Each Task has exactly one Assembly Line Repository, created at the Station where `createAssemblyLineRepo: true` is set.

**Typical Agent workflow with the repository:**

```bash
# Clone the shared workspace
git clone $ASSEMBLY_LINE_REPO_URL workspace
cd workspace

# Do the work (e.g. run analysis, generate code)
# ...
# Write outputs
mkdir -p reports
echo "## Security Report\n..." > reports/security.md

# Commit and push for the next Station
git add reports/security.md
git commit -m "feat(security): add security review"
git push origin main
```

The next Station's Agent clones the same repository and reads `reports/security.md`. The full git history is the audit trail — every file every Agent ever produced is versioned.

---

## Signaling Completion

The Agent signals completion by calling the **`complete_station` MCP tool** exposed by the Operator, then exiting. This is the standard ALP completion mechanism.

### The `complete_station` Tool

Every ALP Operator exposes `complete_station` via its MCP server. The Agent calls it when its work is done:

```
# Example: Claude Code calling complete_station
use_mcp_tool(
  server_name: "alp-operator",
  tool_name: "complete_station",
  arguments: {
    "conclusion": "success",
    "summary": "Security audit complete. Found 2 HIGH severity issues in auth.ts. Report written to reports/security.md.",
    "exitCode": 0
  }
)
```

After calling `complete_station`, the Agent should exit cleanly.

**If the Agent exits without calling `complete_station`:** The Operator's fallback stop hook fires automatically, infers the outcome from the exit state, and calls `complete_station` on the Agent's behalf. This ensures the pipeline always makes progress.

### Exit Codes

| Exit code | Meaning |
|---|---|
| `0` | Success — `complete_station(conclusion: "success")` was called or inferred |
| non-zero | Failure — `complete_station(conclusion: "failure")` was called or inferred |

### Best Practices for Agent Authors

1. **Always call `complete_station` explicitly** before exiting — don't rely on the fallback
2. **Commit and push** all outputs to the Assembly Line Repository before calling `complete_station`
3. **Provide a summary** — it is displayed in the Server UI and helps reviewers at human gates
4. **Exit with code 0** after calling `complete_station` unless there was a fatal error

---

## Agent-Agnosticism

ALP is Agent-agnostic. The Operator can run any AI that:
1. Accepts a prompt (via file, stdin, or CLI argument)
2. Runs as a process
3. Exits with a code when done

**Reference implementation:** Claude Code (`claude --prompt-file initial-prompt.txt`)

**Other potential Agents:**
- OpenAI Codex CLI, GPT-4 via a wrapper CLI
- Gemini CLI
- A custom Python script calling any LLM API
- A human (for manual-step Stations where the Operator just presents a UI)

The choice of Agent type is declared in the Station's `labels` (e.g. `["claude-code"]`). The Operator that handles that label knows how to start and monitor that Agent type.
