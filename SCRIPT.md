# AAP MCP Demo — Presenter Script

> **How to use this script**
> Text in plain paragraphs is what you **say out loud**.
> `Code blocks` are what you **type into the AI assistant**.
> *Italics* are **stage directions** — what to click, show, or watch for.
> Keep a natural pace. Each act has an estimated time.

---

## Pre-demo checklist

*Before the audience arrives:*
- [ ] AI coding assistant open, MCP connected (run `/mcp` if unsure)
- [ ] This repo open in the editor — `playbooks/`, `aap_config/`, `bootstrap_aap.yml` visible
- [ ] Terminal ready with `AAP_PASS` and `AAP_HOST` variables set
- [ ] `oc get csr | grep Pending` returns empty
- [ ] Previous demo cleaned up — no MCP Demo Org in AAP

---

## Act 1 — Show the repo *(~1 minute)*

*Open the file explorer. Walk through the tree as you speak.*

"Before we get into it — one thing worth saying up front. The AAP MCP server is currently in tech preview. That means it's functional, it works, you're going to see it do real things today — but it still has some rough edges and gaps. There are a handful of AAP operations it doesn't expose yet, and we'll hit one of them during the demo. I'll call it out when we get there and show you how we work around it. The point isn't that it's perfect — it's that even at this stage, it changes how you can interact with your automation platform.

Now — let me orient you to what we're working with. Everything you're about to see comes from this git repository — this is the single source of truth for our automation.

In `playbooks/` we have seven Ansible playbooks. They range from a simple hello world all the way to a deliberately misconfigured database setup — you'll see why that one exists in a few minutes.

Over here in `aap_config/` are two YAML files — an inventory definition and a job template definition. This is our config-as-code. AAP will read these files directly from git and configure itself.

And these three playbooks at the root — `bootstrap_aap.yml`, `configure_aap.yml`, and `seed_job_history.yml` — are the orchestration layer. They wire everything together.

The key thing I want you to hold onto: everything the AI assistant does today goes through the AAP MCP server. It's not scripting against a CLI, it's not clicking through a UI — it's calling the AAP API through a set of tools the AI understands natively. Let's see what that looks like."

---

## Act 2 — Foundation via MCP *(~4 minutes)*

"Let's start from zero. Fresh AAP instance, nothing demo-specific configured. The first thing any team needs is a proper organizational structure — org, teams, users, credentials. Watch how fast we can stand this up."

*Switch to the AI assistant.*

"I'll just ask it to tell me what's already there."

```
Check the status of our AAP instance and tell me what's already configured there.
```

*Wait for response. It will show AAP version, existing orgs (Default, APD), baseline templates.*

"Good — we can see the baseline. Default org, the Ansible Product Demos content, nothing from us yet. Now let's build our environment."

```
Create a new organization called "MCP Demo Org" for our demo environment.
```

*Wait for confirmation.*

"One API call. Now teams."

```
Create two teams in MCP Demo Org: "Platform Ops" and "App Developers".
```

*Wait for confirmation.*

"And the users who'll work in those teams."

```
Create two users: ops-user (assign to Platform Ops) and dev-user (assign to App Developers).
```

*Wait for confirmation.*

"Now credentials. We need a machine credential for SSH access and a container registry credential so AAP can pull execution environment images."

```
Create a Machine credential called "MCP Demo SSH Key" and a Container Registry
credential called "MCP Demo Registry" for quay.io — both scoped to MCP Demo Org.
```

*Wait for confirmation.*

"And finally, an execution environment — this is the container image our playbooks will run inside."

```
Create an Execution Environment called "MCP Demo EE" using
registry.redhat.io/ansible-automation-platform/ee-minimal-rhel9:latest,
linked to the MCP Demo Registry credential.
```

*Wait for confirmation.*

"Six operations — organization, two teams, two users, two credentials, one execution environment. No browser, no CLI, no YAML files for any of that. Just natural language. Every single one of those calls went through the MCP server to the AAP API."

---

## Act 3 — Register project + bootstrap *(~2 minutes)*

"Now, there is one thing the MCP server can't do yet — register a git project. Project creation isn't exposed as an MCP tool in this version of AAP. And this is where I want to show you something important about working with AI assistants: when you hit a gap, you don't drop to raw REST calls. You reach for the right tool for the job.

In this case, that's Ansible itself — the same `ansible.controller` collection that AAP uses for its own config-as-code. One playbook, two tasks, fully idempotent."

*Switch to the terminal.*

"I'll run this against the controller directly."

```bash
ansible-playbook bootstrap_aap.yml \
  -e controller_host=$AAP_HOST \
  -e controller_username=admin \
  -e "controller_password=$AAP_PASS"
```

*Wait for playbook to complete — ~30-60 seconds for project sync.*

"Two tasks. First it creates the project and waits for AAP to sync this repo from GitHub. Then it creates a job template called 'Configure AAP' that points at our config-as-code playbook. That job template is the bridge between this git repo and a live AAP instance."

---

## Act 4 — Config-as-code via MCP *(~3 minutes)*

"Here's where it gets interesting. We now have a job template inside AAP that, when launched, will read the YAML files in this repo and configure AAP to match. Let's do that — through the AI assistant."

*Switch back to the AI assistant.*

```
List the available job templates, then launch "MCP Demo | Configure AAP"
and wait for it to complete. Show me what it created.
```

*Wait. The AI will list templates, launch the job, poll until complete (~30s), then list what was created.*

"Watch what just happened. The AI launched a job in AAP. That job ran our `configure_aap.yml` playbook from the synced repo. The playbook read `aap_config/inventories.yml` and `aap_config/job_templates.yml` and created everything defined there — inventory, hosts, nine job templates.

The AI assistant didn't create those templates. AAP configured itself, using this git repo as the source of truth. The AI just pressed the button. That's config-as-code through natural language."

---

## Act 5 — Execute and observe *(~3 minutes)*

"We have templates. Let's run some automation and watch it happen in real time."

```
Launch "MCP Demo | Hello World" passing extra_vars {"operator_name": "Charter Demo"} in the request body.
Wait for it to finish, then show me the full output and individual task events.
```

*Wait for job to complete and output to be displayed.*

"You can see the full Ansible output — exactly what you'd see in the AAP UI, pulled directly through the MCP server. Task by task. Now let's run something a bit more interesting."

```
Now launch "MCP Demo | Environment Report" passing extra_vars {"env_name": "production", "team": "Platform Ops"} in the request body.
```

*Wait for completion.*

"And now I want to ask a question that usually requires digging through audit logs."

```
Show me the activity stream for this session — what operations were performed and by whom?
```

*Wait for response showing the sequence of operations.*

"Every action, timestamped, attributed. That's the full audit trail for everything we've done in this session — org creation, team creation, job launches — all surfaced without leaving this conversation."

---

## Act 6 — Troubleshoot a failed job *(~4 minutes)*

"Now I want to show you what I think is the most powerful part of this demo: how the AI assistant handles failure. Not just reporting it — actually diagnosing it and driving the fix. Watch the full loop."

**Step 1 — Create the failure**

"First, let's create a problem. We have a job template for configuring a database connection. It has a required variable — and I'm going to launch it without providing that variable."

```
Launch "MCP Demo | Configure Database". Don't pass any extra vars — just run it as-is.
```

*Job fails in ~3 seconds.*

"It failed. In a traditional workflow, an operator would now go to the AAP UI, find the job, click into the output, read through the Ansible error, figure out what's missing, go back, edit the job launch, try again. Let's not do that."

**Step 2 — Diagnose**

```
That job failed. Read the output, explain what went wrong, and tell me how to fix it.
```

*Wait for the AI to read stdout, identify the `db_host` assert failure, and explain the fix.*

"It read the job output through the MCP server, found the assert block in the playbook, identified that `db_host` is undefined, and is telling us exactly what variable we need to provide. It didn't need us to copy-paste anything. It went and got the output itself."

**Step 3 — Fix**

"Now fix it."

```
Fix it — launch "MCP Demo | Configure Database" passing extra_vars {"db_host": "db.internal.example.com"} in the request body.
```

*Wait for successful job completion.*

"Job succeeds. Three prompts. No browser, no SSH session, no log grepping. The AI assistant found the root cause, proposed the fix, and executed it — without leaving the conversation."

---

## Act 7 — Understand an existing playbook *(~4 minutes)*

"Let me flip the scenario. Instead of running automation, let's understand automation that's already running. Think about what it means to onboard a new team member, or take over a runbook you've never seen before."

```
Look at the "MCP Demo | Deploy Application" job template. Read the playbook,
check its recent job history, and give me a plain-English summary: what it does,
how often it runs, and any concerns I should know about.
```

*Wait. The AI will read the playbook file, query job history via MCP, surface the ~45% failure rate and change-window policy.*

"It read the playbook source code and cross-referenced it with actual execution history. It found something concerning — this template has a roughly 45% failure rate. Let's dig into that."

```
Why does it fail on production but succeed on development? Walk me through the logic.
```

*Wait for explanation of the change-window assert logic.*

"So it's not a bug — it's a policy enforcement. Production deployments are blocked outside an approved change window. The playbook asserts this at runtime. That's actually good design, but it means whoever's running production deploys needs to understand this.

Let me ask the question a new team member would ask."

```
If I wanted to allow production deployments safely, what would you change?
```

*Wait for response.*

"That's how a new team member gets up to speed in minutes instead of hours. And it's how a senior operator verifies a runbook before running it in production. The knowledge is in the code — the AI just makes it conversational."

---

## Act 8 — Day 2 ops / ChatOps *(~3 minutes)*

"This next act is the NOC use case. Imagine you're on call. Something's been running. You want to know what happened. You don't want to open a browser."

```
What jobs ran in the last hour? Any failures?
```

*Wait for summary.*

"Let's get more specific."

```
Show me all failed jobs. For each one: template name, when it failed, one-line reason.
```

*Wait for the failure list.*

"I can see the Deploy Application production failures we'd expect. Let's rerun one of them against development where it'll succeed."

```
Launch "MCP Demo | Deploy Application" passing extra_vars {"target_env": "development"} in the request body, so it succeeds.
```

*Wait for successful completion.*

"And before I hand off to the next shift, let me check the queue."

```
Which jobs are currently running or pending in the queue?
```

*Wait for response showing empty queue.*

"This replaces 'let me check the dashboard.' You ask, you get the answer in seconds, with drill-down on demand. And you can do this from wherever you're running your AI assistant — your laptop, your IDE, a chat window."

---

## Act 10 — Analytics *(~6 minutes)*

"I've saved the best for last. We have job history — weeks of it, in a real demo you'd have months. Let me show you what happens when you treat your automation data as something you can have a conversation with."

### 10a — System health

```
Give me a health check of our AAP environment — capacity, license usage,
what's running right now, anything concerning.
```

*Wait for metrics summary.*

"Capacity utilization, license consumption, active workloads. That's your environment health in one question."

### 10b — Execution leaderboard

```
Based on our job history, rank all templates by number of runs.
Which is the most executed playbook?
```

*Wait.*

```
Which template has the highest failure rate? Give me percentages, not just counts.
```

*Wait. Deploy Application should surface at ~45%.*

```
Which jobs take the longest? Show average duration by template.
```

*Wait. Data Sync should show ~30s vs ~3-5s for everything else.*

"So we already have something interesting. Data Sync is taking ten times longer than every other template. That's not necessarily a problem — it's a sync job, it's supposed to be slow — but it's visible now. Before you'd need a custom report to see this."

### 10c — Failure pattern deep-dive

```
Look at all the Deploy Application failures. Is there a pattern?
Same error, same environment, same user every time?
```

*Wait. AI will page through failed jobs, read stdout samples, identify: all production, all change-window assertion, all launched by admin.*

"There's the pattern. Every single failure is a production deployment hitting the change-window policy. It's not random — it's structural. Same root cause every time.

Now let me ask the question that bridges technical findings to business impact."

```
What's the business impact of this failure pattern?
What would you recommend to fix it?
```

*Wait for response.*

### 10e — Proactive recommendations

"Final question. And this is the one I always end on."

```
Based on everything you can see — job history, failure rates, resource usage,
RBAC configuration — what are the top three things I should address this week?
```

*Wait for the synthesized recommendation list.*

"This is the difference between a dashboard and an analyst. A dashboard shows you numbers. This tells you what those numbers mean and what to do about it — synthesized across job history, failure patterns, resource data, and access controls, all in one answer.

That's what the AAP MCP server makes possible. Your automation platform, your operational data, accessible through natural language. No dashboards to learn, no APIs to call, no reports to build. Just ask."

---

## Wrap-up *(~1 minute)*

"Let me recap what we just did — in natural language, through an AI assistant, without touching a browser or writing a line of code:

We stood up a complete RBAC structure. We registered a project and ran config-as-code to provision nine job templates. We launched automation, read its output, diagnosed a failure and fixed it on the spot. We audited access controls. And we extracted business intelligence from weeks of job history.

The AAP MCP server is what makes this possible — it's the bridge between the AAP API and the AI assistant. Everything you saw today is available in AAP 4.6 and later.

Questions?"

---

## Appendix — Recovery moves

**If the AI uses `relaunch` instead of `launch` in Act 6:**
> "Let me be more specific — launch the template fresh, not relaunch the previous job, and pass the extra_vars in the request body."

**If a job hangs or times out:**
> "Let me check the status directly." → `What's the current status of job [ID]?`

**If the MCP session has expired:**
> *Run `/mcp` in the AI assistant to reconnect, then repeat the last prompt.*

**If the seeder hasn't run and Act 10 analytics are thin:**
> Skip 10b–10e, focus on 10a (health check) and the troubleshooting story from Act 6.

**If asked "what's the MCP server?":**
> "The Model Context Protocol server is a standardized way for AI assistants to call external APIs as tools. Red Hat ships an MCP server as part of AAP — it exposes the AAP REST API as a set of tools the AI can call directly, with the right schema so the AI knows what each tool does and what parameters it takes."
