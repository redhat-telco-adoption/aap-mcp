# AAP MCP Demo — Notes

## Environment
- AAP 4.7.11 on OpenShift (aap namespace)
- Controller URL: `https://aap-controller-aap.apps.<cluster-domain>/`
- Gateway URL: `https://aap-aap.apps.<cluster-domain>/`
- Admin password: `oc get secret aap-controller-admin-password -n aap -o jsonpath='{.data.password}' | base64 -d`
- MCP URL: `https://aap-mcp-aap.apps.<cluster-domain>/mcp` — token in `.mcp.json` (gitignored)
- Repo: https://github.com/redhat-telco-adoption/aap-mcp (public)

## Baseline (pre-demo — do not delete)
| Resource | ID |
|---|---|
| Org: Default | 1 |
| Org: Ansible Product Demos (APD) | 2 |
| Inventory: Demo Inventory | 1 |
| Inventory: APD Inventory | 2 |
| Project: Demo Project | 6 |
| Project: Ansible Product Demos | 8 |
| Job Template: Demo Job Template | 7 |
| Job Template: APD Single demo setup | 9 |
| Job Template: APD Multi-demo setup | 10 |
| EE: Product Demos EE | 4 |
| Credential: AAP Credential (controller) | 3 |

## Demo-created resources (DELETE between demo runs)

### Gateway/RBAC — created via MCP
| Resource | ID |
|---|---|
| Org: MCP Demo Org | 3 |
| Team: Platform Ops | 1 |
| Team: App Developers | 2 |
| User: ops-user | 5 |
| User: dev-user | 6 |
| Role assignment: ops-user → Platform Ops | 2 |
| Role assignment: dev-user → App Developers | 3 |

### Controller resources — created via MCP + REST API
| Resource | ID |
|---|---|
| Credential: MCP Demo SSH Key | 7 |
| Credential: MCP Demo Registry | 8 |
| EE: MCP Demo EE | 5 |
| Project: MCP Demo Project | 11 |
| JT: MCP Demo \| Configure AAP (bootstrap) | 12 |
| JT: MCP Demo \| Hello World | 13 |
| JT: MCP Demo \| System Info | 14 |
| JT: MCP Demo \| Environment Report | 15 |
| JT: MCP Demo \| Deploy Application (flaky) | 16 |
| JT: MCP Demo \| Data Sync (slow) | 17 |
| JT: MCP Demo \| OS Patching | 18 |
| JT: MCP Demo \| Seed Job History | 19 |
| JT: MCP Demo \| Configure Database (broken) | 20 |
| Inventory: MCP Demo Inventory | 3 |

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

## Act 3 bootstrap commands (REST API — run once per demo)
```bash
AAP_PASS=$(oc get secret aap-controller-admin-password -n aap -o jsonpath='{.data.password}' | base64 -d)
AAP_HOST="https://aap-controller-aap.apps.<cluster-domain>"

# 1. Create project (replace ORG_ID and EE_ID with actual values from Act 2)
PROJECT=$(curl -sk -u "admin:$AAP_PASS" \
  "$AAP_HOST/api/controller/v2/projects/" -H "Content-Type: application/json" \
  -d '{"name":"MCP Demo Project","description":"Demo content from github.com/redhat-telco-adoption/aap-mcp","organization":<ORG_ID>,"scm_type":"git","scm_url":"https://github.com/redhat-telco-adoption/aap-mcp","scm_branch":"main","scm_clean":true,"default_environment":4}')
PROJECT_ID=$(echo $PROJECT | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "Project id: $PROJECT_ID"

# 2. Wait for sync
until curl -sk -u "admin:$AAP_PASS" "$AAP_HOST/api/controller/v2/projects/$PROJECT_ID/" | python3 -c "import sys,json; exit(0 if json.load(sys.stdin)['status']=='successful' else 1)"; do sleep 3; done
echo "Project synced"

# 3. Create bootstrap job template
JT=$(curl -sk -u "admin:$AAP_PASS" \
  "$AAP_HOST/api/controller/v2/job_templates/" -H "Content-Type: application/json" \
  -d "{\"name\":\"MCP Demo | Configure AAP\",\"description\":\"Config-as-code bootstrap\",\"organization\":<ORG_ID>,\"project\":$PROJECT_ID,\"playbook\":\"configure_aap.yml\",\"inventory\":1,\"execution_environment\":4}")
JT_ID=$(echo $JT | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "Bootstrap JT id: $JT_ID"

# 4. Attach AAP Credential
curl -sk -u "admin:$AAP_PASS" \
  "$AAP_HOST/api/controller/v2/job_templates/$JT_ID/credentials/" \
  -H "Content-Type: application/json" -d '{"id":3}'
echo "Credential attached"
```

## Known issues and fixes
1. `configure_aap.yml` — use `vars_files` + `ansible.controller.job_template` directly (not infra.controller_configuration.job_templates role — it silently skips)
2. Job templates reference `Product Demos EE` (id:4) not `MCP Demo EE` (id:5) — MCP Demo EE has no registry password
3. `relaunch` reuses cached project revision — always trigger project sync before relaunching after a code change
4. MCP session expires after ~10 min inactivity → run `/mcp` in Claude Code to reconnect
5. MCP `jobs_stdout_retrieve` only works with `format=json`, not `format=txt`
6. Analytics endpoints (`analytics_retrieve`) require Red Hat Insights subscription — not available; use `metrics_retrieve` + `jobs_list` aggregation instead

## Cleanup script (run between demos)
```bash
AAP_PASS=$(oc get secret aap-controller-admin-password -n aap -o jsonpath='{.data.password}' | base64 -d)
AAP_HOST="https://aap-controller-aap.apps.<cluster-domain>"
AAP_GW="https://aap-aap.apps.<cluster-domain>"

echo "=== Deleting controller job templates ==="
for id in 12 13 14 15 16 17 18 19 20; do
  curl -sk -u "admin:$AAP_PASS" -X DELETE "$AAP_HOST/api/controller/v2/job_templates/$id/" && echo "  Deleted JT $id"
done

echo "=== Deleting controller inventory ==="
curl -sk -u "admin:$AAP_PASS" -X DELETE "$AAP_HOST/api/controller/v2/inventories/3/" && echo "  Deleted inventory 3"

echo "=== Deleting controller project ==="
curl -sk -u "admin:$AAP_PASS" -X DELETE "$AAP_HOST/api/controller/v2/projects/11/" && echo "  Deleted project 11"

echo "=== Deleting controller EE ==="
curl -sk -u "admin:$AAP_PASS" -X DELETE "$AAP_HOST/api/controller/v2/execution_environments/5/" && echo "  Deleted EE 5"

echo "=== Deleting controller credentials ==="
for id in 7 8; do
  curl -sk -u "admin:$AAP_PASS" -X DELETE "$AAP_HOST/api/controller/v2/credentials/$id/" && echo "  Deleted credential $id"
done

echo "=== Deleting gateway users ==="
for id in 5 6; do
  curl -sk -u "admin:$AAP_PASS" -X DELETE "$AAP_GW/api/gateway/v1/users/$id/" && echo "  Deleted user $id"
done

echo "=== Deleting gateway teams ==="
for id in 1 2; do
  curl -sk -u "admin:$AAP_PASS" -X DELETE "$AAP_GW/api/gateway/v1/teams/$id/" && echo "  Deleted team $id"
done

echo "=== Deleting gateway org ==="
curl -sk -u "admin:$AAP_PASS" -X DELETE "$AAP_GW/api/gateway/v1/organizations/3/" && echo "  Deleted org 3"

echo "Cleanup complete. Ready for next demo run."
```
