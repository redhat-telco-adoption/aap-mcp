# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Demo content for Ansible Automation Platform (AAP) controlled through an AI coding assistant via the **AAP MCP server**. The MCP server exposes AAP's REST API as tools the AI can call directly. The demo shows a full operator lifecycle — setup, execution, troubleshooting, RBAC auditing, and analytics — driven by natural language.

## Key commands

### Bootstrap (run once per demo, requires AAP credentials)
```bash
AAP_PASS=$(oc get secret aap-controller-admin-password -n aap -o jsonpath='{.data.password}' | base64 -d)
AAP_HOST="https://aap-controller-aap.apps.<cluster-domain>"

ansible-playbook bootstrap_aap.yml \
  -e controller_host=$AAP_HOST \
  -e controller_username=admin \
  -e "controller_password=$AAP_PASS"
```

### Cleanup (run between demo runs)
```bash
ansible-playbook cleanup_aap.yml \
  -e controller_host=$AAP_HOST \
  -e controller_username=admin \
  -e "controller_password=$AAP_PASS"
```
Then delete gateway resources (org, teams, users) via MCP — `cleanup_aap.yml` only handles controller resources.

### Install collections
```bash
ansible-galaxy collection install -r collections/requirements.yml
```

## Architecture

This repo has two layers that work together:

**1. Ansible playbooks (run locally or from AAP)**
- `bootstrap_aap.yml` — uses `ansible.controller` collection to register the GitHub project in AAP and create the "Configure AAP" job template. Runs locally with `-e controller_host/username/password`. Idempotent.
- `configure_aap.yml` — applies `aap_config/inventories.yml` and `aap_config/job_templates.yml` via `infra.controller_configuration` roles + `ansible.controller.job_template`. This runs *inside AAP* (not locally) after being launched as a job. It reads the CaC YAML files from the synced project.
- `seed_job_history.yml` — builds a flat launch list from `seed_batches` (template + extra_vars + count), then fires all jobs with `wait: false` for parallel execution. Runs inside AAP.
- `cleanup_aap.yml` — removes controller resources (templates, inventory, project, EE, credentials) using `ansible.controller` modules with explicit state=absent loops.

**2. Config-as-code YAML (`aap_config/`)**
- `inventories.yml` — defines `controller_inventories` and `controller_hosts` vars consumed by `infra.controller_configuration` roles in `configure_aap.yml`.
- `job_templates.yml` — defines `controller_job_templates` list consumed by the `ansible.controller.job_template` loop in `configure_aap.yml`. Each entry maps directly to a playbook in `playbooks/`.

**The two-step setup flow:**
1. `bootstrap_aap.yml` (local) → creates project (syncs this repo into AAP) + "MCP Demo | Configure AAP" job template
2. AI assistant launches "MCP Demo | Configure AAP" via MCP → AAP runs `configure_aap.yml` from the synced repo → creates inventory + all 9 demo templates

## MCP tool notes

The AAP MCP server (`mcp__aap__*` tools) wraps the AAP REST API. Critical behavior to know:

- **extra_vars must go in `requestBody`**: `job_templates_launch_create(id=X, requestBody={"extra_vars": {...}})`. Passing `extra_vars` as a top-level keyword argument is silently ignored — the job runs with template defaults.
- **`jobs_relaunch_create` has no `extra_vars` field** — use `job_templates_launch_create` whenever vars need to change.
- **`jobs_stdout_retrieve`** requires `format=json`, not `format=txt`.
- **MCP session expires** after ~10 min inactivity — run `/mcp` to reconnect.

## Playbook design intent

Each playbook in `playbooks/` is intentionally simple and self-contained:
- `04_flaky.yml` — **designed to fail** on `env_name=production` (assert checks for change window). Development succeeds. This creates the analytics failure pattern.
- `05_slow.yml` — uses `ansible.builtin.pause` with a configurable `sleep_seconds` var (~20-30s). This makes it the slowest template in analytics by design.
- `07_misconfigured.yml` — **designed to fail** when `db_host` is not defined. The assert fires immediately (~3s). Used for Act 6 troubleshooting demo.

## Demo sequence dependency

Acts must run in order because each creates resources the next depends on:
Act 2 (gateway resources via MCP) → Act 3 (bootstrap: project + configure JT) → Act 4 (CaC: inventory + demo templates) → seed job history → Acts 5–10 (interactive demos against seeded data)
