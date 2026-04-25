# Trilio Demo Environment — Design and Requirements

## Getting Started

### 1. Copy and populate config files

```bash
cp clouds.yaml.sample clouds.yaml
cp secure.yaml.sample secure.yaml
cp cloud-init-password.yaml.sample cloud-init-password.yaml
cp vars/main.yml.sample vars/main.yml
```

| Sample file | Real file | What to populate |
|-------------|-----------|-----------------|
| `clouds.yaml.sample` | `clouds.yaml` | `auth_url`, `username`, `project_id`, `project_name` |
| `secure.yaml.sample` | `secure.yaml` | OpenStack password |
| `cloud-init-password.yaml.sample` | `cloud-init-password.yaml` | Hashed Cirros console password — generate with `openssl passwd -6 <password>` |
| `vars/main.yml.sample` | `vars/main.yml` | `os_project_id`, `os_project_name`, backup target type IDs |

### 2. Create the keypair

```bash
openstack keypair create vincent-ansible-key --public-key ~/.ssh/id_rsa.pub
```

Or update `vars/main.yml` and the server tasks to reference an existing keypair.

### 3. Install dependencies

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install python-openstackclient ansible openstacksdk
pip install --extra-index-url https://pypi.fury.io/trilio-6-1 workloadmgrclient --no-cache-dir
pip install "urllib3<2"   # avoids LibreSSL warning on macOS
```

### 4. Ensure prerequisite shared infrastructure exists

See [Prerequisite Shared Infrastructure](#prerequisite-shared-infrastructure) below.

---

## Overview

Ansible playbooks that prepare VMs, Trilio workloads, and initial snapshots for
Trilio Backup and Restore demos in OpenStack labs, particularly those created via
Trilio's Lab Manager.

Four playbooks form two symmetric pairs:

```
setup_tenant.yml       ↔  teardown_tenant.yml
setup_trilio.yml       ↔  teardown_trilio.yml
```

---

## Demo Scenarios

All VMs run Cirros — the application names (Firewall, WebApp, Database) are
audience-facing demo labels. Real application images are out of scope for this lab cluster.

| Demo | App Type | Tag | VMs | Purpose |
|------|----------|-----|-----|---------|
| Firewall | Virtual firewall (VNF) | `fw` | 1 | One-Click Restore of a stateful security appliance |
| WebApp | 3-tier web application | `app` | 3 (LB + frontend + backend) | Selective Restore and In-Place Restore demos |
| Database | Primary/replica database | `pg` | 2 | Crash-consistent DB backup; restore across both nodes |

---

## Prerequisite Shared Infrastructure

The following resources must exist in the OpenStack project before running any playbook.
They are **not** created or deleted by these playbooks.

| Resource | Type | Notes |
|----------|------|-------|
| `prod-network` | Network | Primary tenant network for all VMs |
| `data-network` | Network | Secondary network for WebApp BE and Database demos |
| External / provider network | Network | Source of floating IPs |
| `vbsg-ssh` | Security group | Must allow SSH inbound; applied to all VMs |
| `vincent-ansible-key` | Keypair | Injected into all VMs — see setup step 2 |
| S3 backup target | Trilio backup target | Used by Firewall and Database workloads |
| NFS backup target | Trilio backup target | Used by WebApp workload |
| `cirros` | Image | Used for all demo VMs |
| `m1.tiny` | Flavor | Used for all demo VMs |

---

## Resource Naming

Resource names are descriptive by default (e.g., `firewall-vm`, `webapp-lb`). Set
`demo_prefix` in `vars/main.yml` to namespace them (e.g., `"mylab-"` produces
`mylab-firewall-vm`). Teardown only deletes resources matching these names.

### Instances

| Demo | VM Name | Flavor | Network(s) | Floating IP |
|------|---------|--------|------------|-------------|
| Firewall | `firewall-vm` | m1.tiny | prod-network | yes |
| WebApp LB | `webapp-lb` | m1.tiny | prod-network | yes |
| WebApp FE | `webapp-fe` | m1.tiny | prod-network + data-network | no |
| WebApp BE | `webapp-be` | m1.tiny | data-network | no |
| Database Primary | `database-primary` | m1.tiny | prod-network + data-network | yes |
| Database Replica | `database-replica` | m1.tiny | data-network | no |

### Volumes

| Demo | Boot Volume (1G) | Data Volume (4G) |
|------|------------------|------------------|
| Firewall | `firewall-bootvol` | `firewall-datavol` |
| WebApp LB | `webapp-lb-bootvol` | — |
| WebApp FE | `webapp-fe-bootvol` | — |
| WebApp BE | `webapp-be-bootvol` | `webapp-be-datavol` |
| Database Primary | `database-primary-bootvol` | `database-primary-datavol` |
| Database Replica | `database-replica-bootvol` | `database-replica-datavol` |

### Ports

| Demo | Port Name | Network |
|------|-----------|---------|
| Database Primary | `database-primary-data-port` | data-network |

### Trilio Resources

| Demo | Workload Name | Instances | Backup Target | Snapshot Name |
|------|---------------|-----------|---------------|---------------|
| Firewall | `firewall-workload` | firewall-vm | S3 | `firewall-snapshot` |
| WebApp | `webapp-workload` | webapp-lb + webapp-fe + webapp-be | NFS | `webapp-snapshot` |
| Database | `database-workload` | database-primary + database-replica | S3 | `database-snapshot` |

---

## Playbook Design

### `setup_tenant.yml`
Create all demo OpenStack resources from scratch.

**Phase 1 — Prerequisite checks (fail-fast):**
- Networks exist: `prod-network`, `data-network` (project-scoped)
- Security group exists: `vbsg-ssh`
- Keypair exists: `vincent-ansible-key`
- Flavor exists: `m1.tiny`
- Image exists: `cirros`

**Phase 2 — Create resources** (volumes → port → VM, per demo):

| Demo | Resources |
|------|-----------|
| Firewall | boot vol + data vol + VM (prod-network, floating IP) |
| WebApp | 3 boot vols + BE data vol + LB (floating IP) + FE on prod+data (no FIP) + BE on data-network (no FIP) |
| Database | 2 boot vols + 2 data vols + port on data-network + Primary on prod+data (FIP via post-create) + Replica on data-network (no FIP) |

**Phase 3 — Verify:** all VMs reach ACTIVE state.

---

### `teardown_tenant.yml`
Delete all demo OpenStack resources.

1. Delete servers (wait for DELETED before proceeding)
2. Delete data volumes
3. Delete boot volumes
4. Delete ports

Tolerates missing resources throughout — safe to run against a partially-built environment.

---

### `setup_trilio.yml`
Create Trilio workloads and take initial full snapshots.

1. Look up VM IDs by name (project-scoped) for each workload
2. Create workload with those instance IDs and the configured backup target
3. Wait for each workload to reach `available`
4. Trigger a full snapshot per workload
5. Wait for all snapshots to reach `available`
6. Verify all workloads and snapshots are available

---

### `teardown_trilio.yml`
Delete Trilio workloads and snapshots. Supports per-demo tag filtering.

1. List all workloads in project
2. For each targeted demo (fw/app/pg): find matching workload, delete its snapshots, wait, delete workload
3. Running without `--tags` deletes all three workloads

Tolerates missing workloads and snapshots.

---

## Implementation Notes

- **Idempotent**: `state: present/absent` throughout
- **No hardcoded UUIDs**: network lookups use name + `project_id` filter; server/port
  tasks rely on the `cloud:` auth context for project scoping (those modules do not
  accept a `project:` parameter)
- **Auth**: `clouds.yaml` + `secure.yaml`; never source RC files
- **Trilio API**: `openstack workloadmgr` CLI (workloadmgrclient package) — no Ansible
  collection exists for TrilioVault
- **Tags**: `fw`, `app`, `pg` on all tasks — run a single demo with `--tags <demo>`
- **Python interpreter**: tenant playbooks set `ansible_python_interpreter` via
  `VIRTUAL_ENV` env var; trilio playbooks use a hardcoded `openstack_bin` path —
  both ensure the venv Python is used rather than the system Python
- **`OS_CLIENT_CONFIG_FILE`**: set at play level so `openstack.cloud` modules find
  `clouds.yaml` regardless of working directory; `secure.yaml` must be in
  `~/.config/openstack/` for the same reason
- **Boot volumes**: not auto-deleted on VM termination — `teardown_tenant.yml` handles
  explicit deletion
- **Volume mounting**: data volume mount inside VMs is manual (Cirros cloud-init too
  limited for automation)

---

## TODOs

- **Automate data volume mounting** — Firewall, WebApp BE, Database Primary, and
  Database Replica each have a 4G data volume that must be manually mounted inside
  the VM after `setup_tenant.yml` runs. Cirros cloud-init is too limited to handle this
  today; options to explore: switch to a cloud-init-capable image (e.g., minimal
  Alpine/Fedora), or add a post-provision playbook that SSHes in and runs `mkfs` +
  `mount` + `/etc/fstab` entry. Until then, `os-mount-datavol.sh` is scp'd in and
  run by hand.
- **Floating IP access for WebApp FE/BE, Database Replica** — no floating IPs on
  these VMs; reaching them for manual steps requires hopping through another VM in
  the same network. (Database Primary gets a FIP automatically via post-create CLI step.)
- **Snapshot content verification** — `setup_trilio.yml` waits for snapshot `available`
  status only; a deeper check (e.g., snapshot size > 0) would catch silent failures before
  demo day.
- **Inject file before snapshot** — to make restores visually convincing, a known file
  (e.g., a timestamped marker or sample dataset) is manually scp'd into target VMs before
  triggering the snapshot. `setup_trilio.yml` should automate this: SSH into each VM after
  mount, write a recognizable file, then proceed to snapshot.
- **Idempotent re-run of `setup_trilio.yml`** — errors if a workload already exists; add a
  "skip if present" guard so the playbook is safe to re-run against a live environment.

---

## Out of Scope

- Creating shared infrastructure (networks, security groups, images, flavors)
- Installing or configuring Trilio itself
- Volume mount automation inside VMs (tracked in TODOs above)
- HEAT templates
- CI/CD integration

---

## Project Layout

```
.
├── README.md
├── REQUIREMENTS.md
├── ansible.cfg
├── clouds.yaml                       # gitignored — copy from clouds.yaml.sample
├── clouds.yaml.sample
├── secure.yaml                       # gitignored — copy from secure.yaml.sample
├── secure.yaml.sample
├── cloud-init-password.yaml          # gitignored — copy from .sample
├── cloud-init-password.yaml.sample
├── vars/
│   ├── main.yml                      # gitignored — copy from main.yml.sample
│   └── main.yml.sample
└── playbooks/
    ├── setup_tenant.yml
    ├── teardown_tenant.yml
    ├── setup_trilio.yml
    ├── teardown_trilio.yml
    └── tasks/
        └── setup_workload.yml
```
