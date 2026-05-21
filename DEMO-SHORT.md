# AAP MCP Server — Live Demo Script (Short)

## Story
> "Our team just got access to an AAP instance. Using an AI coding assistant and the AAP MCP server,
> we'll set up a complete automation environment, run automation at scale, troubleshoot
> failures, audit access, and extract business intelligence — all through natural language."

*Estimated time: 20–25 minutes*

---

## Act 1 — Show the repo *(30 seconds)*

**Talking point:** "All our automation lives in git. Playbooks, a config-as-code directory,
and a bootstrap playbook that lets AAP configure itself. Everything the AI assistant does today
flows through the AAP MCP server."

Show:
- `playbooks/` — 7 playbooks (hello world through misconfigured db)
- `aap_config/` — inventories.yml + job_templates.yml (CaC)
- `configure_aap.yml` — the bootstrap
- `seed_job_history.yml` — automation that creates automation history

---

## Act 2 — Foundation via MCP

**Prompt 1:**
```
Check the status of our AAP instance and tell me what's already configured there.
Then create a new organization called "MCP Demo Org" for our demo environment.
```

**Prompt 2:**
```
In MCP Demo Org, create two teams: "Platform Ops" and "App Developers".
Then create two users: ops-user (assign to Platform Ops) and dev-user (assign to App Developers).
```

**Prompt 3:**
```
Create a Machine credential called "MCP Demo SSH Key" and a Container Registry
credential called "MCP Demo Registry" for quay.io — both scoped to MCP Demo Org.
```

**Prompt 4:**
```
Create an Execution Environment called "MCP Demo EE" using
registry.redhat.io/ansible-automation-platform/ee-minimal-rhel9:latest,
linked to the MCP Demo Registry credential.
```

**Talking point:** "Every one of those operations went through the MCP server —
no UI, no CLI, just natural language calling the right API."

---

## Act 3 — Register project + bootstrap *(acknowledged MCP gap)*

**Talking point:** "Project creation isn't exposed by the MCP server yet — we drop to Ansible,
the same collection AAP uses internally. One playbook, fully idempotent."

Run the bootstrap playbook (requires `.env` sourced — `source .env`):
```bash
ansible-playbook bootstrap_aap.yml \
  -e controller_host=$AAP_GATEWAY_URL \
  -e controller_username=$AAP_USERNAME \
  -e controller_password=$AAP_GATEWAY_PASS
```

> **Operator note:** Use `AAP_GATEWAY_URL`/`AAP_GATEWAY_PASS`, not `AAP_HOST`/`AAP_CONTROLLER_PASS` —
> the `ansible.controller` collection authenticates through the platform gateway.

---

## Act 4 — Config-as-code via MCP

**Prompt:**
```
List the available job templates, then launch "MCP Demo | Configure AAP"
and wait for it to complete. Show me what it created.
```

**Expected:** Inventory + localhost host + all demo job templates created from the repo's CaC YAML.

---

## Act 4b — Seed job history

**Talking point:** "Before analytics, we need realistic job history. One prompt seeds the whole dataset."

**Prompt:**
```
Launch "MCP Demo | Seed Job History" and wait for it to complete.
```

**What it does:** Fires all jobs in parallel with `wait: false`, building the analytics dataset:

| Template | Runs | Expected failures |
|---|---|---|
| Hello World | 15 | 1 (random) |
| Deploy Application | 11 | 5 (production env → change window assert) |
| Environment Report | 8 | 0 |
| OS Patching | 6 | 0 |
| System Info | 6 | 0 |
| Data Sync | 4 | 0 |

> **Operator note:** The seed job takes ~60–90s (sequential API calls for 48 job launches).
> Child jobs run in the background. Wait ~2 minutes after seed completes before Act 10 analytics.

---

## Act 5 — Execute and observe

**Prompt:**
```
Launch "MCP Demo | Hello World" passing extra_vars {"operator_name": "Demo User"} in the request body.
Wait for it to finish, then show me the full output and individual task events.
```

**Talking point:** "Launch, wait, inspect — all in one prompt. No browser, no SSH."

---

## Act 6 — Troubleshoot a failed job

**Talking point:** "Now let's show how the AI assistant handles failure — not just reporting it,
but diagnosing it and driving the fix. Watch the full loop."

**Step 1 — Create the failure:**
```
Launch "MCP Demo | Configure Database". Don't pass any extra vars — just run it as-is.
```

*(Job fails in ~3 seconds with an assertion error)*

**Step 2 — Diagnose:**
```
That job failed. Read the output, explain what went wrong, and tell me how to fix it.
```

*(The AI assistant reads stdout via MCP, identifies the missing `db_host` variable,
explains the assert block, and proposes the fix)*

**Step 3 — Fix and launch:**
```
Fix it — launch "MCP Demo | Configure Database" passing extra_vars {"db_host": "db.internal.example.com"} in the request body.
```

*(Job succeeds)*

> **Operator note:** Use the word **"launch"**, not "relaunch". `jobs_relaunch_create` has no
> `extra_vars` field — "relaunch" routes to that tool and vars are silently dropped. Also ensure
> extra_vars are passed inside `requestBody`: `{"extra_vars": {...}}`, not as a top-level argument.

**Talking point:** "Three prompts. No browser, no SSH, no log grepping. Root cause found,
fix executed, without leaving the conversation."

---

## Act 8 — Day 2 ops / ChatOps

**Talking point:** "The NOC use case — answers without a browser."

**Prompt:**
```
What jobs ran in the last hour? Any failures? For each failure: template name, when it failed, one-line reason.
```

**Prompt:**
```
Launch "MCP Demo | Deploy Application" passing extra_vars {"target_env": "development"} in the request body, so it succeeds.
```

> **Operator note:** `jobs_list` `search` does a text search — does **not** filter by status.
> `search="failed"` returns zero results. The AI assistant fetches all jobs and filters client-side.

**Talking point:** "This replaces 'let me check the dashboard'. Ask, get the answer, drill down on demand."

---

## Act 9 — RBAC audit

**Talking point:** "Security and compliance teams love this one."

**Prompt:**
```
Give me a complete access report for MCP Demo Org — who has access to what, with their roles.
```

**Prompt:**
```
Are there any users who have been provisioned but never launched a job?
```

**Talking point:** "RBAC audits that used to take an hour now take 30 seconds.
And unlike a static report, you can ask follow-up questions."

---

## Act 10 — Analytics

**Talking point:** "Not a fixed dashboard — a conversation with your automation data."

**Prompt:**
```
Give me a health check of our AAP environment — capacity, license usage,
what's running right now, anything concerning.
```

**Prompt:**
```
Based on our job history, rank all templates by number of runs and failure rate.
Which template has the highest failure rate? Give me percentages, not just counts.
```

**Expected:** Most executed: Hello World (15 runs). Highest failure rate: Deploy Application (45% — 5 of 11).

**Prompt:**
```
Based on everything you can see — job history, failure rates, resource usage,
RBAC configuration — what are the top three things I should address this week?
```

*(Expected: 1) Fix Deploy Application change-window flow — 45% failure rate is operational risk,
2) Data Sync at 30s avg — review for optimization, 3) Provisioned users with no job history — confirm access)*

**Talking point:** "A dashboard shows you numbers. The AI assistant tells you what they mean
and what to do about them. That's the difference."

---

## MCP capability map

| Use case | MCP tools |
|---|---|
| RBAC setup | `organizations_create`, `teams_create`, `users_create`, `role_user_assignments_create` |
| Credentials & EE | `credentials_create`, `execution_environments_create` |
| Project registration | REST API (MCP gap — AI assistant adapts) |
| Config-as-code | `job_templates_launch_create` → AAP configures itself |
| Job launch + observe | `job_templates_launch_create`, `jobs_retrieve`, `jobs_stdout_retrieve`, `jobs_job_events_list` |
| Troubleshooting | `jobs_retrieve` → `jobs_stdout_retrieve` → `job_templates_launch_create` (see note in Act 6) |
| ChatOps | `jobs_list`, `job_templates_launch_create`, `metrics_retrieve` |
| RBAC audit | `role_user_assignments_list`, `users_list`, `teams_list` |
| System health | `metrics_retrieve` |
| Execution analytics | `jobs_list` (paginated) → AI assistant aggregates |

## Analytics dataset (seeded)
| Template | Runs | Failures | Fail% | Avg Duration |
|---|---|---|---|---|
| Hello World | 15 | 1 | 7% | 3s |
| Deploy Application | 11 | 5 | 45% | 4s |
| Environment Report | 8 | 0 | 0% | 5s |
| OS Patching | 6 | 0 | 0% | 5s |
| System Info | 6 | 0 | 0% | 5s |
| Data Sync | 4 | 0 | 0% | 30s |

## Cleanup (run between demos)
Run `ansible-playbook cleanup_aap.yml` to remove all demo-created controller resources.
Gateway resources (org, teams, users) must be deleted manually via MCP or the AAP UI.
