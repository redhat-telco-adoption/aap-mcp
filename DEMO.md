# AAP MCP Server — Live Demo Script

## Story
> "Our team just got access to an AAP instance. Using an AI coding assistant and the AAP MCP server,
> we'll set up a complete automation environment, run automation at scale, troubleshoot
> failures, audit access, and extract business intelligence — all through natural language."

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

**Prompt:**
```
Check the status of our AAP instance and tell me what's already configured there.
```
```
Create a new organization called "MCP Demo Org" for our demo environment.
```
```
Create two teams in MCP Demo Org: "Platform Ops" and "App Developers".
```
```
Create two users: ops-user (assign to Platform Ops) and dev-user (assign to App Developers).
```
```
Create a Machine credential called "MCP Demo SSH Key" and a Container Registry
credential called "MCP Demo Registry" for quay.io — both scoped to MCP Demo Org.
```
```
Create an Execution Environment called "MCP Demo EE" using
registry.redhat.io/ansible-automation-platform/ee-minimal-rhel9:latest,
linked to the MCP Demo Registry credential.
```

**Talking point:** "Every one of those operations went through the MCP server —
no UI, no CLI, just natural language calling the right API."

---

## Act 3 — Register project + bootstrap *(acknowledged MCP gap)*

**Talking point:** "Project creation isn't exposed by the MCP server yet. Rather than dropping
to raw REST calls, we reach for Ansible — the same `ansible.controller` collection that AAP
itself uses for config-as-code. One playbook, two tasks, fully idempotent."

Run the bootstrap playbook (requires controller credentials in env or `~/.netrc`):
```bash
ansible-playbook bootstrap_aap.yml \
  -e controller_host=https://aap-controller-aap.apps.<cluster-domain> \
  -e controller_username=admin \
  -e controller_password=$AAP_PASS
```

What `bootstrap_aap.yml` does:
1. `ansible.controller.project` — creates **MCP Demo Project**, points it at the GitHub repo, waits for sync
2. `ansible.controller.job_template` — creates **MCP Demo | Configure AAP** with the AAP Credential attached

Both tasks are idempotent — safe to re-run between demo runs.

---

## Act 4 — Config-as-code via MCP

**Prompt:**
```
List the available job templates, then launch "MCP Demo | Configure AAP"
and wait for it to complete. Show me what it created.
```

**Expected:** Inventory + localhost host + all demo job templates created from the repo's CaC YAML.

---

## Act 5 — Execute and observe

**Prompt:**
```
Launch "MCP Demo | Hello World" passing extra_vars {"operator_name": "Charter Demo"} in the request body.
Wait for it to finish, then show me the full output and individual task events.
```
```
Now launch "MCP Demo | Environment Report" passing extra_vars {"env_name": "production", "team": "Platform Ops"} in the request body.
```
```
Show me the activity stream for this session — what operations were performed and by whom?
```

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

*(The AI assistant launches the template with `db_host=db.internal.example.com` in the requestBody — job succeeds)*

> **Operator note:** Use the word **"launch"**, not "relaunch". The MCP `jobs_relaunch_create`
> tool schema is missing `extra_vars`, so saying "relaunch" leads the AI to that tool and the
> vars are silently dropped. "Launch" steers it to `job_templates_launch_create`, which accepts
> extra_vars correctly — but only when passed inside `requestBody`: `{"extra_vars": {...}}`.
> Passing `extra_vars` as a top-level argument to that tool is also silently ignored.

**Talking point:** "Three prompts. No browser, no SSH, no log grepping. The AI assistant found
the root cause, proposed the fix, and executed it without leaving the conversation."

---

## Act 7 — Understand an existing playbook

**Talking point:** "Flip it around — instead of running automation, let's understand
what's already there. Institutional knowledge on demand."

**Prompt:**
```
Look at the "MCP Demo | Deploy Application" job template. Read the playbook,
check its recent job history, and give me a plain-English summary: what it does,
how often it runs, and any concerns I should know about.
```

*(The AI assistant reads `playbooks/04_flaky.yml`, queries job history via MCP,
surfaces the 45% failure rate and the change-window policy blocking production)*

```
Why does it fail on production but succeed on development? Walk me through the logic.
```

```
If I wanted to allow production deployments safely, what would you change?
```

**Talking point:** "This is how a new team member gets up to speed in minutes
instead of hours. Or how an operator verifies what they're about to run."

---

## Act 8 — Day 2 ops / ChatOps

**Talking point:** "The NOC use case — answers without a browser."

**Prompt:**
```
What jobs ran in the last hour? Any failures?
```
```
Show me all failed jobs. For each one: template name, when it failed, one-line reason.
```
```
Launch "MCP Demo | Deploy Application" passing extra_vars {"target_env": "development"} in the request body, so it succeeds.
```
```
Which jobs are currently running or pending in the queue?
```

**Talking point:** "This replaces 'let me check the dashboard'. Just ask,
get the answer in seconds, with drill-down on demand."

---

## Act 9 — RBAC audit

**Talking point:** "Security and compliance teams love this one."

**Prompt:**
```
Give me a complete access report for MCP Demo Org — who has access to what,
with their roles.
```
```
What teams does ops-user belong to? What can they actually do?
```
```
Are there any users who have been provisioned but never launched a job?
```
```
Show me all role assignments created in the last 24 hours.
Who granted access to whom?
```

**Talking point:** "RBAC audits that used to take an hour now take 30 seconds.
And unlike a static report, you can ask follow-up questions."

---

## Act 10 — Analytics

**Talking point:** "Finally — the AI analyst angle. Not a fixed dashboard,
but a conversation with your automation data."

### 10a — System health (metrics_retrieve)
```
Give me a health check of our AAP environment — capacity, license usage,
what's running right now, anything concerning.
```

**What the AI assistant surfaces from metrics_retrieve:**
- 1 controller node, 642 capacity slots, 0% utilized
- 100-host license, 2 hosts registered, 98 remaining
- 0 running / 0 pending jobs, system idle

### 10b — Execution leaderboard
```
Based on our job history, rank all templates by number of runs.
Which is the most executed playbook?
```
```
Which template has the highest failure rate? Give me percentages, not just counts.
```
```
Which jobs take the longest? Show average duration by template.
```

**Expected answers:**
- Most executed: Hello World (15 runs)
- Highest failure rate: Deploy Application (45% — 5 of 11 failed)
- Slowest: Data Sync (30s avg) — 10x slower than Hello World (3s avg)

### 10c — Failure pattern deep-dive
```
Look at all the Deploy Application failures. Is there a pattern?
Same error, same environment, same user every time?
```

*(The AI assistant pages through failed jobs, reads stdout samples, finds:
all 5 failures are production deployments, all hit the same "change window" assertion,
all launched by admin)*

```
What's the business impact of this failure pattern?
What would you recommend to fix it?
```

### 10d — Resource utilization
```
Which organization runs more automation — Default or MCP Demo Org?
Break it down by template.
```
```
Are there any job templates that have never been executed?
Flag them as potentially stale.
```

### 10e — Proactive recommendations
```
Based on everything you can see — job history, failure rates, resource usage,
RBAC configuration — what are the top three things I should address this week?
```

*(The AI assistant synthesizes across all data sources and returns a prioritized action list:
1) Fix the Deploy Application change-window flow — 45% failure rate is operational risk,
2) Data Sync runs are slow at 30s — review for optimization opportunities,
3) Two users provisioned with no job history — confirm they still need access)*

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
| ChatOps | `jobs_list`, `jobs_relaunch_create`, `metrics_retrieve` |
| RBAC audit | `role_user_assignments_list`, `users_list`, `teams_list`, `activitystream_list` |
| System health | `metrics_retrieve` |
| Execution analytics | `jobs_list` (paginated) → AI assistant aggregates |
| Failure patterns | `jobs_list` + `jobs_stdout_retrieve` → AI assistant synthesizes |

## Analytics dataset (seeded)
| Template | Runs | Failures | Fail% | Avg Duration |
|---|---|---|---|---|
| Hello World | 15 | 1 | 7% | 3s |
| Deploy Application | 11 | 5 | 45% | 4s |
| Environment Report | 8 | 0 | 0% | 5s |
| OS Patching | 6 | 0 | 0% | 5s |
| System Info | 6 | 0 | 0% | 5s |
| Data Sync | 4 | 0 | 0% | 30s |

## Cleanup (run between demos — see notes.md for full command)
Run `ansible-playbook cleanup_aap.yml` to remove all demo-created controller resources.
Gateway resources (org, teams, users) must be deleted manually via MCP or the AAP UI.
