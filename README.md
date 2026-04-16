# trilio-openstack-ansible

Ansible playbooks that prepare VMs, Trilio workloads, and initial snapshots for
Trilio Backup and Restore demos in OpenStack labs, particularly those created via
Trilio's Lab Manager (not for sale).

The playbooks create a consistent, repeatable demo environment and can be used to
tear it down and rebuild it between demo sessions.

---

## Demo Scenarios

| Demo | Purpose | Restore Time |
|------|---------|-------------|
| **OCR** — One-Click Restore | Delete instance + volumes, restore from snapshot | ~30 min |
| **IR** — In-Place Restore | Restore data volume in place; also used for File Browse | ~2 min |
| **SEL** — Selective Restore | Change network + flavor, then restore original state | ~25 min |
| **DB** — Database Backup | Dual-network and dual-volume instances | — |

---

## Playbooks

```
setup_tenant.yml       ↔  teardown_tenant.yml
setup_trilio.yml       ↔  teardown_trilio.yml
```

| Playbook | What it does |
|----------|-------------|
| `setup_tenant.yml` | Creates VMs, boot volumes, data volumes, and ports |
| `setup_trilio.yml` | Creates Trilio workloads and takes initial full snapshots |
| `teardown_trilio.yml` | Deletes all Trilio workloads and snapshots |
| `teardown_tenant.yml` | Deletes all VMs, volumes, and ports |

All playbooks support `--tags ocr`, `--tags ir`, `--tags sel`, `--tags db` to operate
on a single demo scenario.

---

## Prerequisites

### Shared infrastructure

The following must exist in your OpenStack project before running the playbooks:

| Resource | Notes |
|----------|-------|
| `prod-network` | Primary tenant network |
| `data-network` | Secondary network (DB + SEL demos) |
| External / provider network | Source of floating IPs |
| `vbsg-ssh` security group | Must allow SSH inbound |
| `vincent-ansible-key` keypair | Injected into all VMs — see setup step 3 below |
| Trilio S3 backup target | Used by OCR and DB workloads |
| Trilio NFS backup target | Used by IR and SEL workloads |
| `cirros` image | Used for all demo VMs |
| `m1.tiny` flavor | Used for all demo VMs |

### Software

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install python-openstackclient ansible openstacksdk
pip install --extra-index-url https://pypi.fury.io/trilio-6-1 workloadmgrclient --no-cache-dir
pip install "urllib3<2"   # avoids LibreSSL warning on macOS
```

---

## Setup

### 1. Config files

Copy each sample file and populate it:

```bash
cp clouds.yaml.sample clouds.yaml
cp secure.yaml.sample secure.yaml
cp cloud-init-password.yaml.sample cloud-init-password.yaml
```

| File | What to populate |
|------|-----------------|
| `clouds.yaml` | `auth_url`, `username`, `project_id`, `project_name` |
| `secure.yaml` | OpenStack password |
| `cloud-init-password.yaml` | Hashed Cirros console password — generate with `openssl passwd -6 <password>` |

### 2. Update `vars/main.yml`

- `os_project_id` — your OpenStack project UUID
- `os_project_name` — your project name
- `backup_target_type` values — run `openstack workloadmgr backup target list` to get
  the correct type IDs for your S3 and NFS targets

### 3. Create the keypair

```bash
openstack keypair create vincent-ansible-key --public-key ~/.ssh/id_rsa.pub
```

Or update `vars/main.yml` and the server tasks to reference an existing keypair.

---

## Usage

Always run from the project root directory with the venv active so that `clouds.yaml`
and `secure.yaml` are found:

```bash
source .venv/bin/activate

# Full build
ansible-playbook playbooks/setup_tenant.yml
ansible-playbook playbooks/setup_trilio.yml

# Full teardown + rebuild
ansible-playbook playbooks/teardown_trilio.yml
ansible-playbook playbooks/teardown_tenant.yml
ansible-playbook playbooks/setup_tenant.yml
ansible-playbook playbooks/setup_trilio.yml

# Single demo (e.g. OCR)
ansible-playbook playbooks/teardown_trilio.yml --tags ocr
ansible-playbook playbooks/teardown_tenant.yml --tags ocr
ansible-playbook playbooks/setup_tenant.yml    --tags ocr
ansible-playbook playbooks/setup_trilio.yml    --tags ocr
```

---

## Notes

- All managed resources use the `vincent-` prefix.
- Teardown targets `vincent-*` resources only — shared infrastructure is never touched.
- Boot volumes are not auto-deleted when VMs are removed; delete them explicitly via
  `teardown_tenant.yml` or the OpenStack dashboard.
- Volume mounting inside VMs (for data volumes) is manual — Cirros cloud-init is too
  limited for automation.
- See `REQUIREMENTS.md` for full design detail and constraint documentation.
