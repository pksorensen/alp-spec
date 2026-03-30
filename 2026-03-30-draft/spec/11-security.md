# 11 — Runner Security Model & Credential Architecture

The Runner/Operator boundary is not just an architectural convenience — it is the primary security boundary in ALP. This document explains why that boundary exists, how privileged credentials are managed on the host side of it, and how Agents are given controlled access to external services without ever holding real credentials themselves.

The diagrams above illustrate the sandbox boundary, credential flow, and egress proxy architecture described in each section below.

---

## The Sandbox Boundary

The Runner runs as a privileged process on the **host machine**. It has access to the Docker daemon, host filesystem, network, and any service credentials the operator has registered with it.

The Operator and Agent run **inside a devcontainer** — a Docker container created and destroyed by the Runner for each Job. The container is an isolated sandbox:

- No access to the host filesystem (except explicit bind mounts)
- No Docker daemon access (no socket mount by default)
- No direct network access to internal services
- No knowledge of the real credentials stored on the host

This is the core security property of ALP: **the Agent can only do what the Runner explicitly allows it to do**, because the Runner controls the container's mounts, environment, and network.

```
HOST MACHINE (privileged)
├── ALP Runner (pks-cli)
│   ├── Registered service tokens (Coolify, registry, GitHub, ...)
│   ├── Credential Server  ←── exposes /run/alp/cred.sock
│   └── Egress Proxy       ←── http_proxy for container traffic
│
└── Docker Daemon
    └── DEVCONTAINER (sandboxed) ─────────────────────────────┐
        ├── /run/alp/cred.sock  (bind-mounted from host)      │
        ├── http_proxy=http://host-gateway:3128               │
        ├── Operator (vibecast)                               │
        └── Agent (Claude Code)                               │
                                                              │
        No host filesystem. No Docker socket. No raw tokens. ─┘
```

### Reference Implementation

The current self-hosted runner setup (`docs/self-hosted-runner.md`) demonstrates this boundary in practice:

| Host capability | Container exposure |
|---|---|
| Docker daemon socket | **Not mounted** (agents cannot spawn sibling containers) |
| Coolify API access (same host) | **Not exposed** directly — mediated via credential server |
| GitHub runner token | **Not injected** into container env — only scoped tokens are |
| Host filesystem | **Only workspace bind mount** — no host secrets accessible |

> **Note for pks-cli implementors:** When spawning the devcontainer, do not pass `--privileged` and do not mount `/var/run/docker.sock` unless the Station explicitly declares it needs Docker-in-Docker capability and a human operator has approved that label. The security model depends on the container being unprivileged.

---

## Credential Server

The Runner exposes a **Credential Server** over a Unix socket at `/run/alp/cred.sock` on the host. This path is bind-mounted **read-write** into the devcontainer so the Operator and Agent can reach it:

```yaml
# devcontainer bind mount (added by Runner at job start)
- source: /run/alp/cred.sock
  target: /run/alp/cred.sock
  type: bind
```

The Credential Server is a simple HTTP server over the Unix socket. Clients inside the sandbox call it with `curl --unix-socket /run/alp/cred.sock http://localhost/...`.

### Token Registration

Before Runners start accepting Jobs, the operator registers service credentials with the Runner's configuration:

```yaml
# ~/.pks-cli/runner-credentials.yaml  (host-only, never mounted into containers)
credentials:
  - service: coolify
    token: "eyJ..."          # Coolify API key
    scopes: ["deploy", "read"]
    allowed_labels: ["deploy", "production"]

  - service: registry
    username: "pksorensen"
    token: "ghp_..."         # Container registry PAT
    scopes: ["push", "pull"]
    allowed_labels: ["build", "deploy"]

  - service: github
    token: "ghp_..."         # GitHub PAT for gh CLI
    scopes: ["repo:write", "pr:write"]
    allowed_labels: ["vibecheck", "deliver"]
```

The `allowed_labels` field is the permission gate: a Job must have **all the required labels** to receive a credential for a given service. This is the workload identity check.

### Workload Identity

The credential server authenticates callers by their **workload identity**, not a static secret. When the Runner creates the devcontainer, it injects:

```bash
ALP_JOB_ID=job-abc123          # The current Job ID
ALP_STATION_LABELS=deploy,production   # The Station's declared labels
```

When the Agent calls the credential server, it presents its `ALP_JOB_ID`. The credential server resolves the Job's labels from the Runner's in-memory job registry and checks whether those labels satisfy the `allowed_labels` requirement for the requested service.

This is directly analogous to **GitHub Actions OIDC** / **Azure federated credentials**:

| GitHub Actions OIDC | ALP Credential Server |
|---|---|
| Workflow/job identity → OIDC JWT | Job ID + labels → workload identity |
| OIDC token exchanged for Azure access token | Job identity checked → JIT token issued |
| No static secret in the workflow | No real token in the container env |
| Azure validates the subject claim | Runner validates the label claim |

### Credential Server API

```
GET /token?service=<name>&scopes=<comma-separated>

Response:
{
  "token": "jit_abc...",    // short-lived JIT token (TTL: job lifetime)
  "expires_at": "...",
  "endpoint": "https://registry.example.com"
}

POST /proxy
{
  "url": "https://coolify.example.com/api/v1/deploy",
  "method": "POST",
  "headers": { "Content-Type": "application/json" },
  "body": { "applicationId": "app-123" }
}

Response: proxied response from the target service
(real credential is swapped in by the Runner, never seen by caller)
```

The Agent can use either mode:
- **Token mode** — receive a JIT token and make the API call itself (for CLIs like `gh`, `docker login`)
- **Proxy mode** — let the Runner make the call on its behalf (for one-off HTTP calls where the agent never needs to see the token)

---

## Egress Proxy (DMZ)

The credential server handles credential issuance. The **Egress Proxy** handles all outbound network traffic from the container. Together they form the DMZ between the sandbox and the outside world.

The Runner starts an HTTP/HTTPS proxy (e.g. on `host-gateway:3128`) and injects it into the container:

```bash
# Injected into devcontainer environment by Runner
HTTP_PROXY=http://host-gateway:3128
HTTPS_PROXY=http://host-gateway:3128
```

All HTTP(S) traffic from the Agent (curl, gh CLI, npm, docker, etc.) routes through the Runner's proxy. The proxy can:

1. **Allow or block destinations** — deny access to internal services (e.g. the host's Coolify dashboard) unless the Job has the right labels
2. **Swap bearer tokens** — when the Agent sends a request with a scoped proxy token (`ALP-Proxy-Token: <job-scoped-token>`), the Runner replaces it with the real service credential before forwarding
3. **Log all egress** — every outbound request is logged with the Job ID, enabling full audit trails of what each Agent called
4. **Enforce human gates** — certain destinations (e.g. production Coolify endpoints) require human approval before the request is forwarded

### Token Swap in Detail

The Agent never holds the real Coolify API key. Instead:

1. Agent calls credential server: `GET /token?service=coolify`
2. Runner validates labels, issues a scoped proxy token: `alp_proxy_jit_xxx` (TTL: job lifetime)
3. Agent makes API call: `Authorization: Bearer alp_proxy_jit_xxx` → routed through egress proxy
4. Proxy recognises the `alp_proxy_jit_xxx` prefix, looks up the real credential, replaces the header: `Authorization: Bearer eyJ...real_coolify_key...`
5. Real credential goes to Coolify. Agent never sees it.

If the container is compromised and the `alp_proxy_jit_xxx` token is extracted, the attacker has a token that:
- Only works through the Runner's proxy (not directly against the service)
- Expires when the Job ends
- Is scoped to the service and operation the Job declared

---

## Human-in-the-Loop at the DMZ

> **Diagram:** [diagrams/security-egress-proxy.drawio](../diagrams/security-egress-proxy.drawio)

Certain operations should never be automated without a human check. The Egress Proxy implements **human gates at the network boundary** — not just at the pipeline gate level (see [06-transition-rules.md](06-transition-rules.md)).

A human gate at the DMZ is configured per-destination in the Runner's egress policy:

```yaml
egress_policy:
  - destination: "coolify.example.com/api/v1/deploy"
    method: "POST"
    require_human_approval: true
    notify_channel: "agentics-server"   # ALP Server notifies via UI
    timeout_minutes: 60
```

When a request matching this policy arrives at the proxy:

1. Proxy **holds** the request (does not forward)
2. Proxy notifies the ALP Server: `POST /runners/{id}/approval-requests`
3. The ALP Server shows the pending request in the UI with full request details
4. A human reviews and clicks Approve or Deny
5. ALP Server responds to the Runner: approved/denied
6. Proxy forwards (or drops) the original request

This is fundamentally different from pipeline-level gates:
- **Pipeline gate** (spec/06): pauses the Task between Stations — the Agent has already exited
- **DMZ gate**: intercepts a live in-flight network request — the Agent is still running and waiting for the response

This allows patterns like: *"The deploy station may call the Coolify API, but each individual deploy call requires human approval before it is forwarded."*

---

## What the Agent Sees

From the Agent's perspective, the credential architecture is invisible:

```bash
# Agent requests a credential
curl --unix-socket /run/alp/cred.sock \
  "http://localhost/token?service=registry&scopes=push"
# → {"token": "alp_proxy_jit_xxx", "endpoint": "ghcr.io"}

# Agent uses it normally
echo "alp_proxy_jit_xxx" | docker login ghcr.io -u token --password-stdin
docker push ghcr.io/pksorensen/my-image:latest
# → traffic routes through http_proxy → Runner swaps token → real push succeeds

# Agent has no idea the token was swapped — docker just worked
```

If the credential server is not available (the Job's labels don't permit it), the Agent gets a `403 Forbidden` with a clear error: `"service 'registry' not permitted for labels [vibecheck, analysis]"`.

---

## Security Properties Summary

| Property | How it is achieved |
|---|---|
| Agent cannot access host secrets | Container has no host filesystem mount; no env var with real tokens |
| Agent cannot escalate privileges | No Docker socket mount; container runs unprivileged |
| Credentials are scoped to the Job | Workload identity (Job ID + labels) gates access |
| Credentials expire automatically | JIT tokens have TTL = job lifetime |
| Blast radius if container is compromised | Scoped proxy token + job TTL; no direct service access |
| All egress is auditable | Every outbound request logged with Job ID by proxy |
| Human approval for sensitive operations | DMZ-level gates hold requests until approved |
| No static secrets in container env | Only `ALP_JOB_ID` and scoped proxy tokens injected |

---

## Relationship to Current Implementation

The current implementation (pks-cli + vibecast) partially realises this model. Git credential injection is the first concrete instance of the credential server pattern:

| Concept | Current implementation |
|---|---|
| Credential server | Git credentials injected via `ASSEMBLY_LINE_REPO_TOKEN` env var — first step toward full cred server |
| Workload identity | `ALP_JOB_ID` + `AGENTICS_TOKEN` already injected; label-gating not yet wired |
| Egress proxy | Not yet implemented; planned for pks-cli v2 |
| Human DMZ gate | Implemented at pipeline level (spec/06 gates); DMZ-level planned |
| Sandbox isolation | GitHub Actions devcontainer setup (`docs/self-hosted-runner.md`) provides the boundary today |

See `docs/git-credentials-in-devcontainers.md` for the git credential pattern that is the current reference implementation of the credential injection concept.

---

## Implementing the Credential Server

A compliant ALP Credential Server MUST:

1. Listen on a Unix socket at `/run/alp/cred.sock` on the host
2. Mount the socket into the devcontainer as a bind mount
3. Expose `GET /token` with service/scope parameters
4. Validate callers by `ALP_JOB_ID` against the registered job registry
5. Return `403` for requests where the Job's labels do not satisfy `allowed_labels`
6. Issue JIT tokens scoped to the job lifetime

A compliant ALP Credential Server SHOULD:

1. Support `POST /proxy` for full request proxying with token swap
2. Log all credential issuance with Job ID, service, scopes, and timestamp
3. Revoke all JIT tokens when the Job ends

A compliant ALP Credential Server MAY:

1. Implement egress HTTP/HTTPS proxy with destination allow-listing
2. Implement human-approval gates for sensitive destinations
3. Integrate with external secret stores (Vault, AWS Secrets Manager) rather than flat config files
