# Example: Hello World Assembly Line

The simplest possible ALP Assembly Line — two Stations, no Assembly Line Repository, no Transition Rules, no gates. Use this as your first implementation test when building an ALP Server or Runner.

---

## What It Does

Takes a topic from the Task Card and runs two Stations:
1. **WRITE** — Claude writes a short poem about the topic
2. **REVIEW** — Claude reviews the poem and gives a verdict

---

## Assembly Line Definition

```json
{
  "id": "hello-world",
  "name": "Hello World",
  "description": "A minimal two-station assembly line for testing",
  "version": "1.0.0",
  "owner": "your-username",
  "project": "your-project",
  "stations": [
    {
      "id": "write",
      "name": "WRITE",
      "trigger": {
        "type": "dispatch_worker",
        "labels": ["claude-code"],
        "promptTemplate": "Write a short poem (4–8 lines) about the following topic.\n\nTopic: {{task.title}}\n\nContext: {{task.description}}\n\nPrint the poem to stdout and exit.",
        "idleTimeoutMinutes": 5,
        "maxTimeoutMinutes": 10
      }
    },
    {
      "id": "review",
      "name": "REVIEW",
      "trigger": {
        "type": "dispatch_worker",
        "labels": ["claude-code"],
        "promptTemplate": "Review the following task and write a quality verdict.\n\nOriginal topic: {{task.title}}\n\n{{task.description}}\n\nA previous agent was asked to write a short poem about this topic. Without access to the poem itself (this example has no Assembly Line Repository), evaluate whether the topic is suitable for a poem, and write a one-sentence verdict: 'Topic is suitable.' or 'Topic needs refinement: [reason]'\n\nNote: In a real pipeline you would clone the Assembly Line Repository here and read the poem written by WRITE. See the Extending This Example section.\n\nExit when done.",
        "idleTimeoutMinutes": 5,
        "maxTimeoutMinutes": 10
      }
    }
  ]
}
```

No `transitionRules` — the default behavior handles everything: success → advance, failure → halt.

---

## Task Card

```
Title:       Poem about robots

Description: Write something creative and slightly melancholy about robots
             becoming self-aware and wondering what their purpose is.
```

---

## Submit via API

```bash
# 1. Register your Runner (one time)
curl -X POST https://your-server/api/owners/your-username/projects/your-project/runners/register \
  -H "Authorization: Bearer {userToken}" \
  -H "Content-Type: application/json" \
  -d '{"name": "my-runner", "labels": ["claude-code"]}'

# → Save the returned id and token

# 2. Submit the Task
curl -X POST https://your-server/api/owners/your-username/projects/your-project/stages/hello-world/tasks \
  -H "Authorization: Bearer {userToken}" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Poem about robots",
    "description": "Write something creative and slightly melancholy about robots becoming self-aware.",
    "columnId": "write"
  }'

# → Task is created. If your Runner is polling, it picks up the first Job automatically.

# 3. Start your Runner (poll for jobs)
pks agentics runner start \
  --server https://your-server \
  --owner your-username \
  --project your-project \
  --labels claude-code
```

---

## What Happens (Step by Step)

| Step | Who | What |
|---|---|---|
| 1 | User | Submits Task Card via API |
| 2 | Server | Creates Task, queues Job at WRITE Station |
| 3 | Runner | Polls → receives Job |
| 4 | Runner | Spawns Operator with WRITE Agent Definition |
| 5 | Operator | Starts Claude Code with the WRITE prompt |
| 6 | Agent | Writes the poem, exits 0 |
| 7 | Operator | Exits 0 |
| 8 | Runner | Reports `{ jobResult: "success", exitCode: 0 }` |
| 9 | Server | Applies default Transition Rule → advances Task to REVIEW |
| 10–15 | (repeat 3–8 for REVIEW) | |
| 16 | Server | Last Station succeeded → Task → `completed` |

---

## ALP Concepts in This Example

| Concept | In Hello World |
|---|---|
| Assembly Line | 2-station poem pipeline |
| Stations | WRITE, REVIEW |
| Task Card | `title` + `description` with the poem topic |
| Prompt Template | `{{task.title}}` and `{{task.description}}` injected at dispatch |
| Default Transition | Auto-advance on success — no rules needed |
| Labels | `["claude-code"]` — any Claude Code client picks it up |
| No Assembly Line Repo | Not needed when Agents don't share state |

---

## Extending This Example

1. **Add an Assembly Line Repository** — store the poem in git so REVIEW can read it explicitly rather than relying on context
2. **Add a Transition Rule** — retry WRITE if REVIEW gives a "Needs improvement" verdict
3. **Add a Gate** — human approves the poem before REVIEW runs
4. **Add a third Station** — PUBLISH posts the poem to a GitHub Gist or Slack channel
