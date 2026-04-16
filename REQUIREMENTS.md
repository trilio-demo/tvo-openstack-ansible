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

| Demo | VMs | Volumes | Networks | Purpose |
|------|-----|---------|----------|---------|
| OCR  | 1   | 2       | 1        | One-Click Restore — delete instance + volumes, restore from snapshot |
| IR   | 2   | 3       | 1        | In-Place Restore of data volume; also used for File Browse |
| SEL  | 2   | 1 each  | 2        | Selective Restore — change network + flavor, restore original state |
| DB   | 2   | 3       | 2        | Dual-network and dual-volume instances for DB backup demo |

---

## Prerequisite Shared Infrastructure

The following resources must exist in the OpenStack project before running any playbook.
They are **not** created or deleted by these playbooks.

| Resource | Type | Notes |
|----------|------|-------|
| `prod-network` | Network | Primary tenant network for all VMs |
| `data-network` | Network | Secondary network for DB and SEL demos |
| External / provider network | Network | Source of floating IPs |
| `vbsg-ssh` | Security group | Must allow SSH inbound; applied to all VMs |
| `vincent-ansible-key` | Keypair | Injected into all VMs — see setup step 2 |
| S3 backup target | Trilio backup target | Used by OCR and DB workloads |
| NFS backup target | Trilio backup target | Used by IR and SEL workloads |
| `cirros` | Image | Used for all demo VMs |
| `m1.tiny` | Flavor | Used for all demo VMs |

---

## Resource Naming

All Ansible-managed resources use the `vincent-` prefix. Teardown targets `vincent-*`
resources only — any other resources in the project are never touched.

### Instances

| Demo | VM Name | Flavor | Network(s) |
|------|---------|--------|------------|
| OCR | `vincent-ocr-vm` | m1.tiny | prod-network |
| IR VM1 | `vincent-ir-vm1` | m1.tiny | prod-network |
| IR VM2 | `vincent-ir-vm2` | m1.tiny | prod-network |
| SEL VM1 | `vincent-sel-vm1` | m1.tiny | prod-network |
| SEL VM2 | `vincent-sel-vm2` | m1.tiny | data-network |
| DB1 | `vincent-db1-vm` | m1.tiny | prod-network + data-network |
| DB2 | `vincent-db2-vm` | m1.tiny | prod-network |

### Volumes

| Demo | Boot Volume (1G) | Data Volume (4G) |
|------|------------------|------------------|
| OCR | `vincent-ocr-bootvol` | `vincent-ocr-datavol` |
| IR VM1 | `vincent-ir-vm1-bootvol` | `vincent-ir-vm1-datavol` |
| IR VM2 | `vincent-ir-vm2-bootvol` | — |
| SEL VM1 | `vincent-sel-vm1-bootvol` | — |
| SEL VM2 | `vincent-sel-vm2-bootvol` | — |
| DB1 | `vincent-db1-bootvol` | — |
| DB2 | `vincent-db2-bootvol` | `vincent-db2-datavol` |

### Ports

| Demo | Port Name |
|------|-----------|
| DB1 | `vincent-db1-data-port` |

### Trilio Resources

| Demo | Workload Name | Instances | Backup Target | Snapshot Name |
|------|---------------|-----------|---------------|---------------|
| OCR | `vincent-ocr-workload` | vincent-ocr-vm | S3 | `vincent-ocr-snapshot` |
| IR | `vincent-ir-workload` | vincent-ir-vm1 + vincent-ir-vm2 | NFS | `vincent-ir-snapshot` |
| SEL | `vincent-sel-workload` | vincent-sel-vm1 + vincent-sel-vm2 | NFS | `vincent-sel-snapshot` |
| DB | `vincent-db-workload` | vincent-db1-vm + vincent-db2-vm | S3 | `vincent-db-snapshot` |

---

## Playbook Design

### `setup_tenant.yml`
Create all `vincent-*` OpenStack resources from scratch.

**Phase 1 — Prerequisite checks (fail-fast):**
- Networks exist: `prod-network`, `data-network` (project-scoped)
- Security group exists: `vbsg-ssh`
- Keypair exists: `vincent-ansible-key`
- Flavor exists: `m1.tiny`
- Image exists: `cirros`

**Phase 2 — Create resources** (volumes → port → VM, per demo):

| Demo | Resources |
|------|-----------|
| OCR | boot vol + data vol + VM (prod-network, floating IP) |
| IR | boot vol + data vol + VM1 (floating IP); boot vol + VM2 (no floating IP) |
| SEL | boot vol + VM1 on prod-network (floating IP); boot vol + VM2 on data-network (no floating IP) |
| DB1 | boot vol + port on data-network + VM on prod+data (floating IP) |
| DB2 | boot vol + data vol + VM (prod-network, no floating IP) |

**Phase 3 — Verify:** all VMs reach ACTIVE state.

---

### `teardown_tenant.yml`
Delete all `vincent-*` OpenStack resources.

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
Delete all `vincent-*` Trilio workloads and snapshots.

1. List all workloads; for each: delete all snapshots (wait for completion)
2. Delete each workload (wait for completion)

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
- **Tags**: `ocr`, `ir`, `sel`, `db` on all tasks — run a single demo with `--tags <demo>`
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

- **Automate data volume mounting** — OCR, IR VM1, and DB2 each have a 4G data volume
  that must be manually mounted inside the VM after `setup_tenant.yml` runs. Cirros
  cloud-init is too limited to handle this today; options to explore: switch to a
  cloud-init-capable image (e.g., minimal Alpine/Fedora), or add a post-provision playbook
  that SSHes in and runs `mkfs` + `mount` + `/etc/fstab` entry. Until then,
  `os-mount-datavol.sh` is scp'd in and run by hand.
- **Floating IP access for IR VM2, SEL VM2, DB2** — no floating IPs today; reaching these
  VMs for manual steps requires hopping through another VM in the same network.
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
