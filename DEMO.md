# AAP MCP Server — Live Demo Script

## Story
> "Our team just got access to an AAP instance. Using Claude Code and the AAP MCP server,
> we'll set up a complete automation environment — users, teams, credentials, execution
> environments — and then register our playbook repo and run our first jobs, all through
> natural language."

## Prerequisites
- Claude Code running with the AAP MCP server configured (`.mcp.json`)
- Repo already pushed: `https://github.com/redhat-telco-adoption/aap-mcp`
- AAP admin credentials available

---

## Act 1 — Show the repo (30 seconds)

**Talking point:** "We already have our Ansible content in a git repo. Three playbooks, all running on localhost so the demo is self-contained. We also have config-as-code YAML that describes the inventories and job templates we want AAP to manage."

Show the files:
- `playbooks/01_hello_world.yml`
- `playbooks/02_system_info.yml`
- `playbooks/03_report.yml`
- `aap_config/inventories.yml`
- `aap_config/job_templates.yml`
- `configure_aap.yml` ← the bootstrap playbook that applies all the above to AAP

---

## Act 2 — Build the foundation via MCP

**Prompt Claude Code:**
```
Check the status of our AAP instance and tell me what's already configured there.
```

**Then:**
```
Create a new organization called "MCP Demo Org" for our demo environment.
```

**Then:**
```
Create two teams in the MCP Demo Org: "Platform Ops" and "App Developers".
```

**Then:**
```
Create two users: ops-user (assign to Platform Ops team) and dev-user (assign to App Developers team).
Give them secure passwords and member role assignments.
```

**Then:**
```
Create a Machine credential called "MCP Demo SSH Key" scoped to MCP Demo Org.
Also create a Container Registry credential called "MCP Demo Registry" for quay.io, also scoped to MCP Demo Org.
```

**Then:**
```
Create an Execution Environment called "MCP Demo EE" using the image
registry.redhat.io/ansible-automation-platform/ee-minimal-rhel9:latest,
scoped to MCP Demo Org.
```

**Talking point:** "Every one of those operations went through the MCP server — no UI, no CLI, just Claude Code understanding natural language and calling the right API."

---

## Act 3 — Register the project (honest MCP gap moment)

**Talking point:** "Here's where we hit an interesting limitation. The MCP server exposes a lot of AAP operations, but project creation isn't one of them yet. Watch how Claude Code handles this."

**Prompt Claude Code:**
```
I want AAP to pull our playbooks from https://github.com/redhat-telco-adoption/aap-mcp
Create a project called "MCP Demo Project" in the MCP Demo Org pointing to that repo.
```

Claude Code will note the MCP gap and fall back to direct REST API calls. The commands it runs will look like:

```bash
# Set your AAP controller URL and credentials
export AAP_HOST="https://<your-controller-url>"
export AAP_USER="admin"
export AAP_PASS="<your-password>"

# Create the project
curl -sk -u "$AAP_USER:$AAP_PASS" \
  "$AAP_HOST/api/controller/v2/projects/" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "MCP Demo Project",
    "description": "Demo content from aap-mcp repo",
    "organization": <MCP_DEMO_ORG_ID>,
    "scm_type": "git",
    "scm_url": "https://github.com/redhat-telco-adoption/aap-mcp",
    "scm_branch": "main",
    "scm_clean": true,
    "default_environment": <MCP_DEMO_EE_ID>
  }'

# Create the bootstrap job template
curl -sk -u "$AAP_USER:$AAP_PASS" \
  "$AAP_HOST/api/controller/v2/job_templates/" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "MCP Demo | Configure AAP",
    "description": "Config-as-code bootstrap — creates inventory and job templates",
    "organization": <MCP_DEMO_ORG_ID>,
    "project": <MCP_DEMO_PROJECT_ID>,
    "playbook": "configure_aap.yml",
    "inventory": 1,
    "execution_environment": 4,
    "ask_variables_on_launch": false,
    "extra_vars": ""
  }'
```

> **Note:** `inventory: 1` is "Demo Inventory" (existing) and `execution_environment: 4` is
> "Product Demos EE" (existing, has `infra.controller_configuration` installed).
> The bootstrap job needs the controller credential attached separately:
> `POST /api/controller/v2/job_templates/<id>/credentials/ {"id": 3}`

---

## Act 4 — Config-as-code via MCP

**Talking point:** "Now that the project is registered and synced, we let AAP configure itself. The `configure_aap.yml` playbook uses the `infra.controller_configuration` collection to create our inventory and job templates from the YAML files in the repo."

**Prompt Claude Code:**
```
List the available job templates and then launch the "MCP Demo | Configure AAP" job.
Poll it until it completes and show me the output.
```

**Expected outcome:** Job runs, stdout shows inventory + 3 job templates being created.

---

## Act 5 — Verify via MCP

**Prompt Claude Code:**
```
List all inventories and job templates now. What new resources did the configure job create?
```

**Expected outcome:**
- New inventory: **MCP Demo Inventory** (with localhost host)
- New job templates: **MCP Demo | Hello World**, **MCP Demo | System Info**, **MCP Demo | Environment Report**

---

## Act 6 — Execute and observe

**Prompt Claude Code:**
```
Launch the "MCP Demo | Hello World" job template and wait for it to finish.
Then show me the full output and the individual task events.
```

**Then:**
```
Now launch "MCP Demo | Environment Report" with extra vars: env_name=production, team="Platform Ops"
```

**Then:**
```
Show me the activity stream for the last 15 minutes. What operations were performed and by whom?
```

**Talking point:** "The activity stream is a full audit trail — every resource created, every job launched, every change made. All driven by Claude Code through the MCP server."

---

## Key demo points to emphasize

| What | How |
|------|-----|
| RBAC setup (org, teams, users) | MCP `organizations_create`, `teams_create`, `users_create` |
| Credentials & EE | MCP `credentials_create`, `execution_environments_create` |
| Project registration | Direct REST API (known MCP gap — Claude Code adapts) |
| Config-as-code execution | MCP `job_templates_launch_create` |
| Job launch & observability | MCP `jobs_retrieve`, `jobs_stdout_retrieve`, `jobs_job_events_list` |
| Audit trail | MCP `activitystream_list` |

## Repo structure (reference)
```
aap-mcp/
├── playbooks/
│   ├── 01_hello_world.yml      # greeting + timestamp
│   ├── 02_system_info.yml      # localhost facts
│   └── 03_report.yml           # parameterized health report
├── aap_config/
│   ├── inventories.yml         # CaC: MCP Demo Inventory + localhost host
│   └── job_templates.yml       # CaC: 3 job templates
├── collections/
│   └── requirements.yml        # infra.controller_configuration dependency
└── configure_aap.yml           # bootstrap playbook
```
