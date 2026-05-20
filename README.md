# AAP MCP Demo

Demo content for showing [Ansible Automation Platform](https://www.redhat.com/en/technologies/management/ansible) controlled through an AI coding assistant via the **AAP MCP server** — the Model Context Protocol integration that exposes AAP's REST API as tools an AI can call directly.

The demo covers the full operator lifecycle: initial setup, job execution, failure diagnosis, RBAC auditing, and analytics — all driven by natural language, no browser required.

---

## What this repo contains

```
playbooks/           # 7 demo playbooks (hello world → misconfigured DB)
aap_config/          # Config-as-code: inventories.yml + job_templates.yml
bootstrap_aap.yml    # One-time setup: creates project + Configure AAP job template
configure_aap.yml    # CaC apply: creates inventory + all demo job templates
seed_job_history.yml # Populates ~48 historical jobs for analytics demo
cleanup_aap.yml      # Removes all demo-created controller resources
DEMO.md              # Live demo script — 10 acts with exact prompts
notes.md             # Operator runbook: env details, known issues, commands
```

### Playbooks

| File | Template name | Purpose |
|------|--------------|---------|
| `01_hello_world.yml` | Hello World | Basic launch demo; accepts `operator_name` |
| `02_system_info.yml` | System Info | Gathers EE facts |
| `03_report.yml` | Environment Report | Health report; accepts `env_name`, `team` |
| `04_flaky.yml` | Deploy Application | **Fails on production** (change-window assert) — failure analytics |
| `05_slow.yml` | Data Sync | ~30s sleep — duration analytics |
| `06_patching.yml` | OS Patching | Multi-task realistic scenario |
| `07_misconfigured.yml` | Configure Database | **Fails without `db_host`** — troubleshooting demo |

---

## Prerequisites

- AAP 4.6+ with the MCP server enabled and a bearer token
- `ansible-playbook` 2.15+
- Collections: `ansible.controller >=4.5.0`, `infra.controller_configuration >=2.9.0`
- `oc` CLI (if retrieving the admin password from OpenShift)

Install collections:
```bash
ansible-galaxy collection install -r collections/requirements.yml
```

Configure the MCP server in your AI coding assistant's `.mcp.json`:
```json
{
  "mcpServers": {
    "aap": {
      "type": "http",
      "url": "https://<aap-mcp-host>/mcp",
      "headers": { "Authorization": "Bearer <token>" }
    }
  }
}
```

---

## Setup (run once per demo)

### 1. Bootstrap — create the project and Configure AAP template

```bash
AAP_PASS=$(oc get secret aap-controller-admin-password -n aap \
  -o jsonpath='{.data.password}' | base64 -d)

ansible-playbook bootstrap_aap.yml \
  -e controller_host=https://aap-controller-aap.apps.<cluster-domain> \
  -e controller_username=admin \
  -e "controller_password=$AAP_PASS"
```

This creates **MCP Demo Project** (synced from this repo) and the **MCP Demo | Configure AAP** job template. Both are idempotent — safe to re-run.

### 2. Run the demo

Follow `DEMO.md`. Acts 2–10 are driven entirely through the AI assistant via MCP. The only shell commands needed are the bootstrap above and (optionally) the seeder.

---

## Demo structure

| Act | What happens | How |
|-----|-------------|-----|
| 1 | Show the repo | Visual walkthrough |
| 2 | Create org, teams, users, credentials, EE | AI → MCP |
| 3 | Register project, create Configure AAP template | `bootstrap_aap.yml` |
| 4 | Apply config-as-code (inventory + all templates) | AI → MCP → AAP |
| 5 | Launch Hello World + Environment Report, read output | AI → MCP |
| 6 | Trigger failure → diagnose → fix | AI → MCP |
| 7 | Understand an existing playbook from job history | AI → MCP |
| 8 | Day-2 ops / ChatOps queries | AI → MCP |
| 9 | RBAC audit | AI → MCP |
| 10 | Analytics: leaderboard, failure patterns, recommendations | AI → MCP |

### Seeded analytics dataset

Run `MCP Demo | Seed Job History` after Act 4 to populate historical data:

| Template | Runs | Failures | Fail% | Avg Duration |
|----------|------|----------|-------|--------------|
| Hello World | 15 | 1 | 7% | 3s |
| Deploy Application | 11 | 5 | 45% | 4s |
| Environment Report | 8 | 0 | 0% | 5s |
| OS Patching | 6 | 0 | 0% | 5s |
| System Info | 6 | 0 | 0% | 5s |
| Data Sync | 4 | 0 | 0% | 30s |

---

## Cleanup

```bash
ansible-playbook cleanup_aap.yml \
  -e controller_host=https://aap-controller-aap.apps.<cluster-domain> \
  -e controller_username=admin \
  -e "controller_password=$AAP_PASS"
```

Then delete the gateway resources (org, teams, users) via MCP or the AAP UI — these are not managed by `cleanup_aap.yml`.

---

## Known issues

1. **extra_vars in MCP launch calls** — pass them inside `requestBody`: `{"extra_vars": {...}}`. A top-level `extra_vars` kwarg is silently ignored. Phrase prompts as *"passing extra_vars `{...}` in the request body"*.
2. **`jobs_relaunch_create` drops extra_vars** — the tool schema omits the field. Use `job_templates_launch_create` whenever you need to change variables.
3. **`jobs_stdout_retrieve`** — only works with `format=json`, not `format=txt`.
4. **MCP session expiry** — the session expires after ~10 min of inactivity. Run `/mcp` in the AI assistant to reconnect.
5. **Kubelet CSR approval (SNO clusters)** — pending CSRs cause `remote error: tls: internal error` on all AAP jobs. Check before each run: `oc get csr | grep Pending`.

See `notes.md` for the full operator runbook.
