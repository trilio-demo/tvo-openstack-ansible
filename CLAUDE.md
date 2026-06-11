# CLAUDE.md — Trilio Demo Environment (Canonical clone)

**This clone targets:** Canonical OpenStack (presales TVO).
Branch `canonical`. See `docs/new-cluster-bringup.md` for the active bringup
checklist.

## Session Continuity

This user closes VSCode and CC CLI frequently and **never rejoins sessions**. Every new
conversation starts cold. When you detect a session-end signal, sync memory before
responding with a farewell.

**Session-end phrases:** "that's it for the day", "good session, nite", "stopping point",
or any clear sign-off.

**What to sync:** update `project_trilio_demos.md` with playbook status, pending tasks,
and any new decisions or findings from the session. If this is a per-cluster clone,
also update the Session State section at the bottom of this file.

---

## Project Purpose

Ansible playbooks to build, teardown, and rebuild a Trilio-for-OpenStack demo
environment on the OpenStack cluster identified below. Replaces ad-hoc bash scripts.
See [REQUIREMENTS.md](REQUIREMENTS.md).

**Multi-cluster pattern:** one clone per cluster. See
[docs/new-cluster-bringup.md](docs/new-cluster-bringup.md) for the bringup playbook.

---

## Environment

Cluster-specific identity (Keystone endpoint, project/user/domain IDs,
network names, backup targets, Trilio versions, cluster-host access) lives in
the gitignored **`CLAUDE.local.md`** — it's customer-identifying and must not
be committed. Claude Code auto-loads it at cold start.

Structural notes that are safe to commit:
- **Auth:** OS_* env vars from `vars/main.yml` + `vars/vault.yml`.
  `validate_certs: false` + `--insecure` on every workloadmgr call (self-signed
  + IP endpoint). See [docs/auth-design.md](docs/auth-design.md).
- **openrc:** `vars/openrc/<cluster>.sh` (gitignored; sourced manually for CLI
  work, **not** consumed by playbooks at runtime). Template `*.sh.sample`.
- **Region casing** is environment-specific (this clone uses `RegionOne`;
  RHOSO used `regionOne`) — read it from `vars/main.yml`, don't assume.

## Tool Environment

- First started: Claude Code CLI
- Date: 2026-05-22

---

## Playbook Priority Order

```
setup_trilio.yml       ↔  teardown_trilio.yml
setup_tenant.yml       ↔  teardown_tenant.yml
```

**`setup_tenant.yml`** — create all demo OpenStack resources (VMs, volumes, ports)

**`teardown_tenant.yml`** — delete demo VMs, volumes, ports

**`setup_trilio.yml`** — create Trilio workloads + initial snapshots against running VMs

**`teardown_trilio.yml`** — delete workloads + snapshots (supports per-demo `--tags`)

---

## Critical Constraints

- **Never hardcode UUIDs.** Always look up by name + `project: "{{ os_project_id }}"`.

- **Network names are environment-specific.** Defined in `vars/main.yml` as
  `prod_network` / `data_network`. Both must be **tenant-owned**, not
  shared/admin-owned: `setup_tenant.yml` filters by `project_id` and
  treats a shared-but-visible network as missing. The external
  (floating-IP) network is discovered dynamically via `--external` and
  *is* expected to be shared.

- **Volume quota: ~20 free recommended for first run.** Demos use 10
  volumes (2 firewall + 4 webapp + 4 database); Trilio snapshot
  ephemerals add a similar number during initial backup. Re-runs reuse
  existing resources so headroom matters mostly on fresh bringup.
  `setup_tenant.yml` Phase 1 reads `max_total_volumes` /
  `total_volumes_used` and prints an **advisory** (non-fatal). Raise
  with `openstack quota set --volumes <N> <project>` if a fresh bringup
  would exceed headroom.

- **Do not use `trilio.tiny`.** Defined with 10G disk (wrong spec). Use `m1.tiny`
  (default) or `trilio.small`.

- **Teardown scope is demo resources only.** Stale resources (`vbst_*`, `vince-*`,
  `demo-*`) are left alone. Never delete shared infra (networks, security groups,
  keypairs, images, backup targets).

- **Trilio API via CLI.** No Ansible collection for OpenStack TrilioVault.
  Use `openstack workloadmgr` CLI commands (workloadmgrclient pip package).

- **`workloadmgr` ignores `verify: false` in clouds.yaml.** For self-signed clouds,
  `--insecure` must be passed on every workloadmgr invocation. Playbooks bake it
  into `openstack_bin` so every shell call gets it.

- **Dual-NIC servers cannot use `auto_ip` or `openstack.cloud.floating_ip`.**
  Both fail with "No port matching NAT destination network". Assign floating IPs
  via CLI: `openstack server add floating ip --fixed-ip-address <ip> <server> <fip>`.

- **`workloadmgr workload show -f json` returns empty stdout.** Use
  `workload list -f json` + `selectattr` filter for status polling instead.

- **Volume mounting is manual.** Cirros cloud-init too limited. `os-mount-datavol.sh`
  is scp'd in and run by hand.

---

## Resource Naming

`demo_prefix` in `vars/main.yml` defaults to `""`. Set to e.g. `"vincent-"` to
namespace. Names below show the default (no prefix).

| Demo | VMs | Volumes | Port |
|------|-----|---------|------|
| Firewall | `firewall-vm` | `firewall-bootvol` (1G), `firewall-datavol` (4G) | — |
| WebApp | `webapp-lb`, `webapp-fe` (dual-NIC), `webapp-be` | `webapp-{lb,fe,be}-bootvol` (1G), `webapp-be-datavol` (4G) | — |
| Database | `database-primary` (dual-NIC, FIP post-create), `database-replica` | `database-{primary,replica}-bootvol` (1G), `database-{primary,replica}-datavol` (4G) | `database-primary-data-port` |

| Resource | Name |
|----------|------|
| Workloads | `firewall-workload`, `webapp-workload`, `database-workload` |
| Snapshots | `firewall-snapshot`, `webapp-snapshot`, `database-snapshot` |
| Keypair | `vincent-ansible-key` (independent of `demo_prefix`) |

---

## Ansible Conventions

- Use `openstack.cloud` collection (not legacy `os_*` modules)
- **Never** pass `cloud:` to openstack.cloud tasks — auth flows through play-level
  `environment:` block setting `OS_AUTH_URL`, `OS_USERNAME`, `OS_PASSWORD`, etc.
  See [docs/auth-design.md](docs/auth-design.md) for the full rationale.
- Always pass `project: "{{ os_project_id }}"` (or filter by it) on OpenStack lookups
- `vars/main.yml` — cluster-specific non-secret vars (auth_url, username, domains,
  project, workload defs)
- `vars/vault.yml` — `vault_os_password` (gitignored; encrypt with ansible-vault when
  ready)
- Trilio CLI: `openstack workloadmgr` commands via workloadmgrclient pip package
- Teardown: use `failed_when: false` on delete tasks; tolerate missing resources
- Wait loops: `retries` + `delay` on info tasks; not `pause`
- CLI flag: `--workload_id` (underscore, not hyphen)

---

## Running the Playbooks

```bash
# Activate venv (run from Demos/ directory)
source .venv/bin/activate

# Full teardown + rebuild
ansible-playbook playbooks/teardown_trilio.yml
ansible-playbook playbooks/teardown_tenant.yml
ansible-playbook playbooks/setup_tenant.yml
ansible-playbook playbooks/setup_trilio.yml

# Single demo (e.g., Firewall)
ansible-playbook playbooks/teardown_trilio.yml --tags fw
ansible-playbook playbooks/teardown_tenant.yml --tags fw
ansible-playbook playbooks/setup_tenant.yml    --tags fw
ansible-playbook playbooks/setup_trilio.yml    --tags fw
```

---

## Adjacent investigations & Session State

These live in the gitignored **`CLAUDE.local.md`** (auto-loaded at cold start),
because they're full of customer-identifying cluster internals:

- **Adjacent investigations** — the Horizon-slowness two-layer story (infra-team
  interactions, neutron/OVN internals, the redeploy ask, Slack/GitHub links).
- **Session State** — the forward-looking brief (open items, next-session pickup,
  last-session headline) plus the thread-by-thread completion log.

If `CLAUDE.local.md` is missing (e.g. a fresh clone), there's no Session State
to read — treat it as a new bringup and see `docs/new-cluster-bringup.md`.

Full archaeology: `docs/session-state.md` — consult when prior-thread depth,
decision reasoning, or ruled-out paths are needed.
