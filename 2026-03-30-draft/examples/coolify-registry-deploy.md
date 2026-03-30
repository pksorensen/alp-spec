# Example: Credential-Gated Deploy to Coolify + Container Registry

This example shows a two-station Assembly Line that builds and publishes a Docker image to a container registry, then deploys it via Coolify — without the Agent ever holding a real API key.

It demonstrates the full ALP credential architecture described in [spec/11-security.md](../spec/11-security.md).

The swimlane diagram above shows the complete flow from image build through to the human-gated Coolify deploy.

---

## Setup

The Runner runs on the **same Ubuntu 24.04 host** as Coolify (matching the setup in `docs/self-hosted-runner.md`). This means the Runner has inherent network access to Coolify's local API. The credential architecture ensures Agents cannot abuse this access.

### Runner Credential Configuration

```yaml
# ~/.pks-cli/runner-credentials.yaml   (never mounted into containers)

credentials:
  - service: registry
    endpoint: ghcr.io
    username: pksorensen
    token: "${GHCR_TOKEN}"          # ghp_ PAT with packages:write
    scopes: ["push", "pull"]
    allowed_labels: ["build", "deploy"]

  - service: coolify
    endpoint: https://coolify.internal/api/v1
    token: "${COOLIFY_API_TOKEN}"   # Coolify API key
    scopes: ["deploy", "read"]
    allowed_labels: ["deploy"]
    # production deploys require human approval at the DMZ level:
    require_approval_for_paths: ["/api/v1/deploy", "/api/v1/restart"]

egress_policy:
  # Allow: package registry, GitHub API
  - destination: "ghcr.io"
    allow: true
  - destination: "api.github.com"
    allow: true
  # Require approval: Coolify deploy endpoints
  - destination: "coolify.internal/api/v1/deploy"
    method: POST
    require_human_approval: true
  # Block everything else by default
  - destination: "*"
    allow: false
```

---

## Assembly Line: BUILD → DEPLOY

```json
{
  "name": "Build and Deploy",
  "description": "Build a Docker image, push to ghcr.io, deploy via Coolify.",
  "columns": [
    {
      "id": "build",
      "name": "BUILD",
      "order": 0,
      "trigger": {
        "type": "dispatch_worker",
        "promptTemplate": "Build and push the Docker image for this deployment.\n\n{{task.description}}\n\nSteps:\n1. Read the task description to find: repository URL, image name, tag\n2. Clone the repository\n3. Get registry credentials: curl --unix-socket /run/alp/cred.sock 'http://localhost/token?service=registry&scopes=push'\n4. Log in to the registry using the returned token\n5. Build the Docker image: docker build -t <image>:<tag> .\n6. Push the image: docker push <image>:<tag>\n7. Write the image digest to workspace/image-digest.txt\n8. Call complete_station with conclusion: success",
        "appendSystemPrompt": "You are a build engineer. Use the ALP credential server at /run/alp/cred.sock to get registry credentials — never use hardcoded tokens. Build the image exactly as specified. Write the pushed image digest to workspace/image-digest.txt before completing.",
        "labels": ["build", "docker"],
        "idleTimeoutMinutes": 5,
        "maxTimeoutMinutes": 30
      }
    },
    {
      "id": "deploy",
      "name": "DEPLOY",
      "order": 1,
      "trigger": {
        "type": "dispatch_worker",
        "promptTemplate": "Deploy the built image via Coolify.\n\n{{task.description}}\n\nSteps:\n1. Read workspace/image-digest.txt to get the image digest\n2. Read the task description to find: Coolify application ID, environment\n3. Get a deploy token: curl --unix-socket /run/alp/cred.sock 'http://localhost/token?service=coolify&scopes=deploy'\n4. Trigger the deployment via the ALP egress proxy:\n   curl -X POST https://coolify.internal/api/v1/deploy \\\n     -H 'Authorization: Bearer <jit-token>' \\\n     -H 'Content-Type: application/json' \\\n     -d '{\"applicationId\": \"<app-id>\", \"imageDigest\": \"<digest>\"}'\n   NOTE: The proxy will hold this request for human approval before forwarding.\n5. Wait for the 200 response (it will come after a human approves the deploy)\n6. Log the deployment URL from the response\n7. Call complete_station with conclusion: success and summary including the deploy URL",
        "appendSystemPrompt": "You are a deployment engineer. Use the ALP credential server for all credentials. Your deploy call will be held for human approval — this is expected, just wait for the response. Never bypass the proxy. Record the deployment URL in your summary.",
        "labels": ["deploy"],
        "idleTimeoutMinutes": 60,
        "maxTimeoutMinutes": 90
      }
    }
  ],
  "transitionRules": [
    { "fromColumnId": "build", "toColumnId": "deploy", "condition": "success" }
  ]
}
```

---

## The Credential Flow in Detail

### Step 1 — BUILD station: registry token

The Agent inside the BUILD devcontainer calls the credential server:

```bash
# Inside the devcontainer (sandbox)
CRED=$(curl -s --unix-socket /run/alp/cred.sock \
  "http://localhost/token?service=registry&scopes=push")

TOKEN=$(echo $CRED | jq -r .token)
ENDPOINT=$(echo $CRED | jq -r .endpoint)

# Token is a scoped JIT token: alp_proxy_jit_aaa...
# Endpoint is: ghcr.io

# Log in — docker traffic routes through http_proxy → Runner swaps token
echo "$TOKEN" | docker login $ENDPOINT -u token --password-stdin

# Runner's proxy intercepts the docker login request,
# recognises alp_proxy_jit_aaa, replaces with real GHCR_TOKEN,
# forwards to ghcr.io. Agent never saw the real token.

docker build -t ghcr.io/pksorensen/my-app:${GIT_SHA} .
docker push ghcr.io/pksorensen/my-app:${GIT_SHA}
```

**What each party holds:**

| Party | Token it has | Can it reach ghcr.io directly? |
|---|---|---|
| Runner (host) | Real `GHCR_TOKEN` (ghp_...) | Yes — but doesn't expose it |
| Agent (container) | `alp_proxy_jit_aaa` (JIT, TTL=job) | Only through the proxy |
| Proxy (Runner) | Swaps on the wire | Forwards with real token |

---

### Step 2 — DEPLOY station: Coolify deploy with human gate

The DEPLOY station Agent requests a deploy token and calls the Coolify API:

```bash
# Inside the devcontainer
CRED=$(curl -s --unix-socket /run/alp/cred.sock \
  "http://localhost/token?service=coolify&scopes=deploy")
JIT_TOKEN=$(echo $CRED | jq -r .token)

IMAGE_DIGEST=$(cat workspace/image-digest.txt)

# This request goes: Agent → Egress Proxy → [HELD] → Human Approver → Coolify
# The Agent just makes a normal HTTP call and waits:
RESPONSE=$(curl -s -X POST https://coolify.internal/api/v1/deploy \
  -H "Authorization: Bearer $JIT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"applicationId\": \"app-abc123\", \"imageDigest\": \"$IMAGE_DIGEST\"}")
```

**What happens at the proxy:**

```
Agent POST /api/v1/deploy  ──► Egress Proxy
                                    │
                          policy: require_human_approval=true
                                    │
                          ──► ALP Server: POST /runners/{id}/approval-requests
                                    │     {url, method, body, jobId, labels}
                                    │
                          ALP Server shows in UI ──► Human reviews
                                    │
                          Human clicks "Approve"
                                    │
                          ──► Proxy receives approval
                                    │
                          Proxy swaps JIT token → real COOLIFY_API_TOKEN
                                    │
                          ──► Coolify API (real request forwarded)
                                    │
                          ◄── 200 OK {deployUrl: "https://..."}
                                    │
                          ◄── Agent receives response (after ~seconds of human review)
```

The Agent's `curl` call simply blocks until the proxy resolves the approval. From the Agent's perspective, the call just took longer than usual.

---

## Task Card Example

```
Title: Deploy my-app v2.3.1 to production

Description:
  Repository:    https://github.com/pksorensen/my-app
  Branch:        main
  Git SHA:       a1b2c3d
  Image name:    ghcr.io/pksorensen/my-app
  Tag:           2.3.1
  Coolify App:   app-abc123
  Environment:   production
  Notes:         Includes the new auth refactor — review the deploy carefully.
```

---

## Why Not Just Give the Agent the API Key?

| Approach | Risk if agent is compromised |
|---|---|
| Agent has real Coolify API key in env | Attacker can deploy anything, delete apps, read secrets |
| Agent has real registry token in env | Attacker can push malicious images to the registry |
| ALP credential server (this example) | Attacker has a JIT token that expires when the job ends, only works through the proxy, and can only trigger the specific operations the Station's labels permit |

The ALP credential model is directly inspired by how GitHub Actions OIDC works with Azure: the CI job never holds a static cloud credential. Its identity (workload identity) is used to request a short-lived, scoped access token at runtime. If the runner environment is compromised, the blast radius is bounded.

---

## Current Implementation Status

This example represents the **target architecture**. The current state:

| Feature | Status |
|---|---|
| Runner on Coolify host | ✅ Implemented (`docs/self-hosted-runner.md`) |
| Coolify access from runner | ✅ Same host, network reachable |
| Registry credentials (manual) | ✅ Set as GitHub Actions secrets |
| ALP Credential Server socket | ⬜ Not yet implemented in pks-cli |
| Egress Proxy with token swap | ⬜ Not yet implemented |
| Human DMZ gate | ⬜ Planned for pks-cli v2 |
| Workload identity label gating | ⬜ Partially implemented (Job ID injected; label check not wired) |

Today, the pattern is approximated by injecting credentials as env vars (e.g. `ASSEMBLY_LINE_REPO_TOKEN`). The credential server is the next evolution, narrowing the blast radius significantly.
