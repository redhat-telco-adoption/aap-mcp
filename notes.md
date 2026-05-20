# AAP MCP Demo — Notes

## Environment
- AAP 4.7.11 on OpenShift (aap namespace)
- Controller URL: `https://aap-controller-aap.apps.<cluster-domain>/`
- Gateway URL: `https://aap-aap.apps.<cluster-domain>/`
- Admin password: `oc get secret aap-controller-admin-password -n aap -o jsonpath='{.data.password}' | base64 -d`
- MCP URL: `https://aap-mcp-aap.apps.<cluster-domain>/mcp` — URL and token in `.env` (gitignored)
- Repo: https://github.com/redhat-telco-adoption/aap-mcp (public)

> Before launching Claude Code: `source .env` — required for MCP auth (`AAP_MCP_URL`, `AAP_TOKEN`)

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
- Broken JT: id:41 "MCP Demo | Configure Database"
- Failure mode: `assert db_host is defined` fails immediately (3.4s)
- Fix: launch with `{"db_host": "db.internal.example.com"}` passed in request body
- Successful fixed job: id:181 (status: successful, 9.4s)
- **MCP gap / operator tips (two rules):**
  1. Say **"launch"** not "relaunch". `jobs_relaunch_create` has no `extra_vars` field —
     vars are silently dropped. "Launch" steers the AI to `job_templates_launch_create`.
  2. Always phrase extra_vars as **"passing extra_vars `{...}` in the request body"**.
     Passing `extra_vars` as a keyword argument to `job_templates_launch_create` is also
     silently ignored — only `requestBody={"extra_vars": {...}}` is accepted by the MCP tool.

## Act 3 bootstrap commands (run once per demo)
```bash
AAP_CONTROLLER_PASS=$(oc get secret aap-controller-admin-password -n aap -o jsonpath='{.data.password}' | base64 -d)
AAP_HOST="https://aap-controller-aap.apps.<cluster-domain>"

ansible-playbook bootstrap_aap.yml \
  -e controller_host=$AAP_HOST \
  -e controller_username=admin \
  -e controller_password=$AAP_CONTROLLER_PASS
```

`bootstrap_aap.yml` uses `ansible.controller.project` (with `wait: true`) and
`ansible.controller.job_template` — no raw REST calls, fully idempotent.

## Known limitations

- Bootstrap/cleanup playbooks use `ansible.controller` collection which targets the Controller API (`/api/v2/`) directly — not the gateway. This requires separate controller credentials (`AAP_CONTROLLER_PASS`) distinct from the gateway token (`AAP_TOKEN`). The newer `ansible.platform` collection (v2.7+) and `infra.aap_configuration` (v4.5+) support gateway-native auth but migration is deferred.

## Known issues and fixes
1. `configure_aap.yml` — use `vars_files` + `ansible.controller.job_template` directly (not infra.controller_configuration.job_templates role — it silently skips)
2. Job templates reference `Product Demos EE` (id:4) not `MCP Demo EE` (id:5) — MCP Demo EE has no registry password
3. `relaunch` reuses cached project revision — always trigger project sync before relaunching after a code change
4. MCP session expires after ~10 min inactivity → run `/mcp reconnect aap` in the AI coding assistant to reconnect
5. MCP `jobs_stdout_retrieve` only works with `format=json`, not `format=txt`
6. MCP `job_templates_launch_create` extra_vars — must be inside `requestBody={"extra_vars": {...}}`. A top-level `extra_vars` kwarg is silently ignored; the job runs with template defaults. Confirmed 2026-05-20 (jobs 235–237).
7. Analytics endpoints (`analytics_retrieve`) require Red Hat Insights subscription — not available; use `metrics_retrieve` + `jobs_list` aggregation instead
8. **Kubelet CSR approval (SNO cluster)** — this cluster does not auto-approve kubelet serving certificate rotation. Pending CSRs cause `remote error: tls: internal error` on all pod log/exec/stdin streaming, which breaks every AAP automation job. Before each demo run, check and approve:
   ```bash
   oc get csr | grep Pending   # should be empty
   oc get csr -o name | xargs oc adm certificate approve   # approve if any are pending
   ```
   Symptoms: AAP jobs fail in ~4s with `Receptor detail: Error streaming stdin to pod … remote error: tls: internal error`. `oc logs` on any pod in the `aap` namespace returns the same error.

## Cleanup script (run between demos)
```bash
AAP_CONTROLLER_PASS=$(oc get secret aap-controller-admin-password -n aap -o jsonpath='{.data.password}' | base64 -d)
AAP_HOST="https://aap-controller-aap.apps.<cluster-domain>"

ansible-playbook cleanup_aap.yml \
  -e controller_host=$AAP_HOST \
  -e controller_username=admin \
  -e "controller_password=$AAP_CONTROLLER_PASS"
```
