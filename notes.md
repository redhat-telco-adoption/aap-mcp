# AAP MCP Demo — Notes

## Environment
- AAP 4.7.11 on OpenShift (aap namespace)
- Controller URL: `https://aap-controller-aap.apps.<cluster-domain>/`
- Gateway URL: `https://aap-aap.apps.<cluster-domain>/`
- Admin password: `oc get secret aap-controller-admin-password -n aap -o jsonpath='{.data.password}' | base64 -d`
- MCP URL: `https://aap-mcp-aap.apps.<cluster-domain>/mcp` — token in `.mcp.json` (gitignored)
- Repo: https://github.com/redhat-telco-adoption/aap-mcp (public)

## Baseline (pre-demo — do not delete)
- Org: Default
- Org: Ansible Product Demos (APD)
- Inventory: Demo Inventory, APD Inventory
- Project: Demo Project, Ansible Product Demos
- Job Templates: Demo Job Template, APD Single demo setup, APD Multi-demo setup
- EE: Product Demos EE
- Credential: AAP Credential (controller)

> IDs vary per cluster — look them up with `ansible.controller` or the MCP tools if needed.

## Demo-created resources (DELETE between demo runs)

### Gateway/RBAC — created via MCP
- Org: MCP Demo Org
- Teams: Platform Ops, App Developers
- Users: ops-user, dev-user
- Role assignments: ops-user → Platform Ops, dev-user → App Developers

### Controller resources — created via bootstrap_aap.yml + configure_aap.yml
- Credentials: MCP Demo SSH Key, MCP Demo Registry
- EE: MCP Demo EE
- Project: MCP Demo Project
- Job Templates: MCP Demo | Configure AAP, Hello World, System Info, Environment Report, Deploy Application, Data Sync, OS Patching, Seed Job History, Configure Database
- Inventory: MCP Demo Inventory

## Analytics dataset (seeded — ~57 jobs)
| Template | Runs | Failures | Fail% | Avg Duration |
|---|---|---|---|---|
| Hello World | 15 | 1 | 7% | 3s |
| Deploy Application | 11 | 5 | 45% | 4s |
| Environment Report | 8 | 0 | 0% | 5s |
| OS Patching | 6 | 0 | 0% | 5s |
| System Info | 6 | 0 | 0% | 5s |
| Data Sync | 4 | 0 | 0% | 30s |

## Troubleshooting demo (Act 6)
- Broken JT: id:20 "MCP Demo | Configure Database"
- Failure mode: `assert db_host is defined` fails immediately (3.4s)
- Fix: relaunch with `db_host=db.internal.example.com`
- Successful relaunch job id after fix: TBD (run during demo)

## Act 3 bootstrap commands (run once per demo)
```bash
AAP_PASS=$(oc get secret aap-controller-admin-password -n aap -o jsonpath='{.data.password}' | base64 -d)
AAP_HOST="https://aap-controller-aap.apps.<cluster-domain>"

ansible-playbook bootstrap_aap.yml \
  -e controller_host=$AAP_HOST \
  -e controller_username=admin \
  -e controller_password=$AAP_PASS
```

`bootstrap_aap.yml` uses `ansible.controller.project` (with `wait: true`) and
`ansible.controller.job_template` — no raw REST calls, fully idempotent.

## Known issues and fixes
1. `configure_aap.yml` — use `vars_files` + `ansible.controller.job_template` directly (not infra.controller_configuration.job_templates role — it silently skips)
2. Job templates reference `Product Demos EE` (id:4) not `MCP Demo EE` (id:5) — MCP Demo EE has no registry password
3. `relaunch` reuses cached project revision — always trigger project sync before relaunching after a code change
4. MCP session expires after ~10 min inactivity → run `/mcp` in the AI coding assistant to reconnect
5. MCP `jobs_stdout_retrieve` only works with `format=json`, not `format=txt`
6. Analytics endpoints (`analytics_retrieve`) require Red Hat Insights subscription — not available; use `metrics_retrieve` + `jobs_list` aggregation instead

## Cleanup script (run between demos)
```bash
AAP_PASS=$(oc get secret aap-controller-admin-password -n aap -o jsonpath='{.data.password}' | base64 -d)
AAP_HOST="https://aap-controller-aap.apps.<cluster-domain>"

ansible-playbook cleanup_aap.yml \
  -e controller_host=$AAP_HOST \
  -e controller_username=admin \
  -e "controller_password=$AAP_PASS"
```
