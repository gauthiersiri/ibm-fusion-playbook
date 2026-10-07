# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Project

Ansible playbook to configure OpenShift clusters and install IBM Storage Fusion.
Stack: Ansible 2.14+, `kubernetes.core` collection (≥2.4), `oc` CLI in PATH.

## Running the playbook

```bash
source ~/ansible-venv/bin/activate
ansible-galaxy collection install -r requirements.yml

# With kubeconfig (cert-based auth — no Bearer token)
ansible-playbook site.yml \
  -e ocp_api_url=https://api.<cluster>:6443 \
  -e ocp_kubeconfig=~/.kube/config \
  -e ocp_entitlement_key=<key> \
  -e ocp_validate_certs=false

# Run only prerequisites (skip operator install)
ansible-playbook site.yml ... --tags prerequisites

# Skip pull secret (already configured)
ansible-playbook site.yml ... --tags prerequisites --skip-tags pull_secret
```

## Core principles (always apply)

- **Use Ansible modules, not CLI commands.** Prefer `kubernetes.core.k8s` / `kubernetes.core.k8s_info` over `oc` shell commands. Use `ansible.builtin.*` modules over `ansible.builtin.command` / `ansible.builtin.shell` whenever a module exists for the task (file, copy, tempfile, uri, etc.).
- **Always read the IBM documentation before implementing.** Never rely on memory or assumptions about IBM Fusion resource names, namespaces, channels, or procedures — they differ from expectations. Key docs:
  - Pull secret: https://www.ibm.com/docs/en/fusion-software/2.14.0?topic=prerequisites-creating-image-pull-secret
  - CatalogSource: https://www.ibm.com/docs/en/cloud-paks/rhos/1.0.0?topic=clusters-adding-operator-catalog
  - Operator deploy CLI: https://www.ibm.com/docs/en/fusion-software/2.14.0?topic=fusion-deploying-using-cli
  - Use `mcp__product-knowledge__search`, `mcp__propel-web-search__*`, or `mcp__exa__*` tools to fetch docs when IBM pages are behind cookie walls.

## Critical non-obvious rules

### Authentication (`roles/prerequisites/tasks/auth.yml`)
- Auth produces `_k8s_auth` dict fact, NOT `_ocp_token`. Every `kubernetes.core.*` task must use:
  ```yaml
  host: "{{ _k8s_auth.host }}"
  api_key: "{{ _k8s_auth.api_key | default(omit) }}"
  kubeconfig: "{{ _k8s_auth.kubeconfig | default(omit) }}"
  validate_certs: "{{ _k8s_auth.validate_certs }}"
  ```
- Client-cert kubeconfigs (no Bearer token) set `kubeconfig:` path instead of `api_key:`. Both paths must be handled — `oc whoami --show-token` fails on cert-based auth.

### Pull secret (`roles/prerequisites/tasks/pull_secret.yml`)
- IBM doc: patch the **global** `openshift-config/pull-secret`, **not** a namespace-scoped secret.
- The task reads the existing secret, checks if `cp.icr.io` is already in `auths{}`, and **skips the patch if present** — idempotent.
- Patching triggers an OCP MachineConfig rollout (all nodes restart one by one) — expected.
- Use `kubernetes.core.k8s_info` + `kubernetes.core.k8s` only — no `oc` commands.

### Catalog source (`roles/prerequisites/tasks/catalog_source.yml`)
- Only **one** CatalogSource needed: `ibm-operator-catalog` with `icr.io/cpopen/ibm-operator-catalog:latest`.
- No separate Fusion-specific catalog image. Version is selected via OLM channel only.
- Task checks if the CatalogSource already exists and **skips creation** if found (cluster pre-installed it).

### Fusion Subscription (`roles/fusion_operator/tasks/subscription.yml`)
- IBM doc operator package name is `isf-operator` (NOT `ibm-spectrum-fusion`).
- OperatorGroup name: `isf-og`. Subscription name: `isf-operator`.
- `fusion_approval_strategy` defaults to `Manual` — IBM explicitly warns against `Automatic`.
- `source` points to `ibm-operator-catalog` (the single catalog), NOT a fusion-specific one.

### Version/channel mapping
- `fusion_version: "2.12.4"` → auto-derives `fusion_operator_channel: "v2.12"` via Jinja2 in `group_vars/`.
- Channel pattern: `v<MAJOR>.<MINOR>` (e.g. `v2.12`, not `stable-2.12`).

### ansible.cfg
- The file must NOT start with `---` (YAML marker) — it is INI format; `---` causes a parse error.
