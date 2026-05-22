# New Cluster Bringup

How to adapt these playbooks to a new OpenStack cluster.

> **NOTE for the Claude session reading this on a cold start:** If the
> working directory contains `tvo-canonical` and the branch is `canonical`,
> skip to **[Current bringup in progress](#current-bringup-in-progress)**
> below — there's an in-flight checklist you should pick up. The generic
> playbook starts after that section.

---

## Current bringup in progress

**Cluster:** Canonical OpenStack (presales TVO)
**Branch:** `canonical` (created off `realistic-apps`)
**Target clone path:** `~/Development/Lab/tvo-canonical/Demos`
**Decision date:** 2026-05-22
**Status:** Auth chain validated end-to-end against Canonical Keystone
on 2026-05-22. `vars/main.yml` fully populated except `backup_target_type`
IDs (one per workload). Remaining work: verify pre-existing project
resources (keypair, SG, image, default backup target), discover
backup_target_type IDs, then end-to-end validation of
`setup_tenant.yml` → `setup_trilio.yml` → teardown → rebuild for
idempotency.

### Reference material captured

- openrc: `vars/openrc/canonical.sh` (gitignored, in-repo). Originally
  sourced from `~/Development/Lab/tvo-canonical/RC/vince-demo-openrc.sh`,
  but per repo self-containment rule the canonical copy lives in-repo.
- Keystone: `https://172.22.12.33:5000/v3` (self-signed, IP — keep
  `validate_certs: false` + `--insecure`)
- Project: `vince-demo` / `870e66a32c564ed1be66006c1ee5906d`
- User: `vince`
- User domain: `admin_domain` (not `Default`)
- Project domain: openrc supplies `OS_PROJECT_DOMAIN_ID=9291da41e5394e9a89b0ab74e63d6f13`.
  Working assumption: name is also `admin_domain` (Horizon login domain).
  Verify with `openstack --insecure domain show 9291da41e5394e9a89b0ab74e63d6f13 -f value -c name`.
- Region: `RegionOne` (note casing — RHOSO was `regionOne`)

### Checklist

- [x] Decision: clone-per-cluster + fresh `uv` venv per clone
- [x] `docs/new-cluster-bringup.md` written (this file)
- [x] `CLAUDE.md.sample` template added so new clones can `cp` it on day one
- [x] Branch `canonical` created off `realistic-apps` and pushed
- [x] Repo cloned into `~/Development/Lab/tvo-canonical/Demos`
- [x] `CLAUDE.md` in new clone created from `CLAUDE.md.sample` and edited
      for Canonical (Environment section, Session State)
- [x] Fresh `uv venv` created with ansible-core + openstack.cloud collection
      (+ `workloadmgrclient` 6.1.1.11 from Trilio's Gemfury index;
      `pytz` + `docutils` installed manually as undeclared transitive deps)
- [x] Openrc imported into repo as `vars/openrc/canonical.sh` (gitignored,
      mode 0600); sanitized template added at `vars/openrc/canonical.sh.sample`;
      `.gitignore` updated to exclude `vars/openrc/*.sh` and allow `.sh.sample`
- [x] `vars/main.yml` created from sample, updated for Canonical auth,
      region, domains (network names + backup_target_type IDs still `<TBD>`)
- [x] `vars/main.yml.sample` rewritten to OS_* shape (was stale —
      pre-`7dd9aad` migration template)
- [x] `vars/vault.yml` populated with Canonical password for user `vince`
- [x] `cloud-init-password.yaml` created from sample, hashed cirros console
      password populated (used by `setup_tenant.yml` as VM userdata —
      surfaced when first VM-create task ran)
- [x] Project domain name verified — `admin_domain` (assumption from
      Horizon login domain was correct; confirmed implicitly by auth_test
      passing with that value)
- [x] Tenant network names: `prod_network: prod-network`,
      `data_network: data-network` (`data-network` added mid-bringup
      after `setup_tenant.yml`'s project-scope check rejected the
      shared `private-network`; `public-network` is the external/FIP
      network, auto-discovered by plays)
- [x] `ansible.cfg` cleaned: removed dangling `inventory =` directive
      (no playbooks use a host inventory; all use `hosts: localhost`)
- [x] Prereqs verified in project via `playbooks/discover.yml`:
      keypair `vincent-ansible-key`, SG `vbsg-ssh`, boot images
      (Ubuntu 20.04, cirros, Trilio-FRM-Ubuntu-24.04), backup targets
      (NFS default + S3), backup-target-types (`NFS_LAB_CANONICAL`
      default, `S3_LAB`)
- [x] `backup_target_type` wired in `vars/main.yml`: firewall=`S3_LAB`,
      webapp=`NFS_LAB_CANONICAL`, database=`S3_LAB` (mirrors original
      demo intent of exercising both target types)
- [x] `playbooks/auth_test.yml` passed against Canonical (2026-05-22)
- [x] `ansible.cfg` warnings squelched: `localhost_warning=False` (in
      `[defaults]`) + `inventory_unparsed_warning=False` (in `[inventory]`)
- [x] Volume quota advisory added to `setup_tenant.yml` Phase 1
      (non-fatal debug message; advises ~20 free volumes for first run)
- [x] `setup_tenant.yml` + `setup_trilio.yml` validated end-to-end
      against Canonical (2026-05-22): all VMs/volumes provisioned, all
      three Trilio workloads created with their initial snapshots
      completing successfully
- [ ] Teardown + rebuild validated (idempotency) — **deferred**
      (session-end decision 2026-05-22). Setup flow validated end-to-end,
      but the full teardown→rebuild cycle was not run. Will surface if
      a future session needs to rebuild the demo from scratch.
- [x] CLAUDE.md Session State updated: bringup complete 2026-05-22
- [x] Project memory updated to record Canonical bringup completed
      (see [[project-canonical-bringup]])

Mark each box `[x]` as you go. When all boxes are checked, move this
section down to **History of bringups** at the bottom with a one-line
summary and start the next cluster's checklist (if any) here.

---

## Pattern: one clone per cluster

Each cluster gets its **own clone**, in its own directory, on its own branch,
with its own venv. The clones share git history through the public repo but
have independent working trees so `vars/main.yml`, `CLAUDE.md`, and any
cluster-specific tweaks stay stable per cluster.

```
~/Development/Lab/
├── osp17/Demos/            ← branch: realistic-apps  (RHOSO)
├── tvo-canonical/Demos/    ← branch: canonical       (Canonical)
└── <next-cluster>/Demos/   ← branch: <slug>
```

The parent directory name (`osp17`, `tvo-canonical`, …) is the cluster
identifier. The `osp17/` path is a legacy holdover from when that clone
targeted RHOSP 17 — it now holds RHOSO. **Treat the parent directory name
as a hint, not a contract; the branch + `vars/main.yml` are authoritative.**

## Why not branches in a single clone?

Tried that mentally. Three problems:
- `vars/main.yml` would churn every time you switch clusters.
- `vars/vault.yml` (gitignored) holds *one* password; switching clusters
  silently re-uses the wrong one.
- The venv lives next to the working tree, so one clone = one venv, which
  matches one cluster's tooling versions.

Worktrees would solve the first two, but the venv-per-cluster ergonomics
of a fresh clone beat the disk-space savings.

## Bringup checklist (new cluster)

Run this top-to-bottom. Each section calls out what to look up on the new
cluster and what to write into the playbooks.

### 1. Repo + branch + clone

```bash
# In an existing clone (any branch), create the new cluster branch and push.
git checkout -b <cluster-slug>     # e.g. canonical, ibm-cloud-2027
git push -u origin <cluster-slug>

# Clone fresh into the new cluster's directory.
mkdir -p ~/Development/Lab/<cluster-slug>
cd ~/Development/Lab/<cluster-slug>
git clone -b <cluster-slug> https://github.com/trilio-demo/tvo-openstack-ansible.git Demos
cd Demos
```

### 2. Venv

Standard now is `uv`. The venv must have ansible-core, the openstack
collection, and the openstack + workloadmgr clients.

```bash
uv venv
source .venv/bin/activate

# Standard packages (public PyPI).
uv pip install 'ansible-core>=2.16' openstacksdk python-openstackclient

# Trilio's workloadmgrclient is NOT on public PyPI — it lives on a private
# Gemfury index, one per Trilio release line (trilio-6-1, trilio-6-2, ...).
# Match the index to the cluster's installed Trilio release. For Trilio 6.1.x:
uv pip install --extra-index-url https://pypi.fury.io/trilio-6-1 \
  workloadmgrclient --no-cache-dir

# workloadmgrclient has undeclared transitive deps. Install them too,
# or `workloadmgr --help` fails with ModuleNotFoundError and
# `openstack workloadmgr` floods "Could not load" warnings:
uv pip install pytz docutils

ansible-galaxy collection install openstack.cloud
```

If the index returns 403, the URL may need a token:
`https://<token>@pypi.fury.io/trilio-6-1`. The `trilio-6-1` index is
publicly readable as of 2026-05-22.

Sanity:
```bash
ansible --version
openstack --version
openstack workloadmgr --help | head -5   # no "Could not load" lines
workloadmgr --help | head -3              # exits cleanly, no traceback
```

### 3. Auth — import the cluster's openrc into the repo

The repo is self-contained: per-cluster openrcs live **inside** the repo
under `vars/openrc/`, not in some parent directory. Download the openrc
from the cluster (Horizon → API Access → Download OpenStack RC File) and
import:

```bash
cp /path/to/<cluster>-openrc.sh vars/openrc/<cluster>.sh
chmod 600 vars/openrc/<cluster>.sh
```

`vars/openrc/*.sh` is gitignored; `vars/openrc/*.sh.sample` is the only
checked-in artifact (a generic template). The openrc is NOT consumed by
the playbooks at runtime — it's a reference asset for manual CLI work
during bringup and debugging, and the source we copy auth values from
into `vars/main.yml` below.

Capture these OS_* values from your imported openrc:

| Field | Where it lives | RHOSO example | Canonical example |
|---|---|---|---|
| `OS_AUTH_URL` | openrc | `https://keystone-public-…/` | `https://172.22.12.33:5000/v3` |
| `OS_PROJECT_ID` | openrc / `openstack project show` | `6f030e47…` | `870e66a3…` |
| `OS_PROJECT_NAME` | openrc | `vince-demo` | `vince-demo` |
| `OS_USER_DOMAIN_NAME` | openrc | `Default` | `admin_domain` |
| `OS_PROJECT_DOMAIN_NAME` | openrc / `openstack domain show <id>` | `Default` | resolve from `_ID` |
| `OS_REGION_NAME` | openrc | `regionOne` | `RegionOne` |

**Gotcha:** Some clouds (e.g. Canonical) set `OS_PROJECT_DOMAIN_ID` in the
openrc instead of `_NAME`. Current play templates consume `_NAME`, so
resolve once:

```bash
source vars/openrc/<cluster>.sh
openstack --insecure domain show <id> -f value -c name
```

A deferred follow-up (not in any bringup — its own change) is to make
plays accept either `_NAME` or `_ID` so this lookup isn't needed. Until
then, do the lookup. Don't edit play `environment:` blocks during a
bringup — that risks regressing the `7dd9aad` osp17→RHOSO auth refactor.

**Gotcha:** Region casing varies (`regionOne` vs `RegionOne`). Copy from
openrc verbatim.

### 4. Edit `vars/main.yml`

Replace the auth block at the top with the new cluster's values:

```yaml
os_auth_url: <new>
os_username: <new>
os_project_id: <new>
os_project_name: <new>
os_user_domain_name: <new>
os_project_domain_name: <new>
os_region_name: <new>
os_interface: public
```

### 5. Edit `vars/vault.yml`

```bash
cp vars/vault.yml.sample vars/vault.yml
# edit vault_os_password to the new cluster's password for this user
```

`vars/vault.yml` is gitignored — never commit. (If you encrypt it with
`ansible-vault`, that's still the same file path, just encrypted.)

### 5b. Create `cloud-init-password.yaml`

`setup_tenant.yml` injects a cirros console password into VMs via
cloud-init userdata. The hash-only file is gitignored; `.sample` template
shows the shape.

```bash
read -sp 'Choose Cirros console password: ' P; echo
HASH=$(openssl passwd -6 "$P"); unset P
sed "s|<hashed-password>|$HASH|" cloud-init-password.yaml.sample > cloud-init-password.yaml
```

Pipes the plaintext into `openssl passwd -6` (sha512-crypt) and writes
only the hash into the file. The plaintext never enters shell history.

### 6. Network names

`prod_network` / `data_network` in `vars/main.yml` must match real networks
in the new project. The external (floating-IP) network is discovered
dynamically via `--external` — no edit needed.

```bash
openstack --insecure network list --project $OS_PROJECT_ID
```

Pick the two tenant networks the demo VMs should attach to and put their
exact names (case-sensitive!) into `vars/main.yml`.

### 7. Pre-existing project infrastructure

These resources are **assumed to exist** in the project before any play runs.
They are not created by these playbooks. Run:

```bash
ansible-playbook playbooks/discover.yml
```

It prints the project's keypairs, security groups, images, backup targets,
and **backup-target-types** (the latter is what `backup_target_type` in
`vars/main.yml` references — a separate entity from `backup target list`).

Verify the project contains:

| Resource | Default name | What `discover.yml` shows |
|---|---|---|
| Keypair | `vincent-ansible-key` | `=== KEYPAIRS ===` block |
| Security group | `vbsg-ssh` | `=== SECURITY GROUPS ===` block |
| Boot image | (referenced in `setup_tenant.yml`) | `=== IMAGES ===` block |
| Trilio default backup target | one with `Is_Default=true` | `=== BACKUP TARGETS ===` block |
| Trilio backup-target-type | for each workload's `backup_target_type` | `=== BACKUP TARGET TYPES ===` block |

Also check the project's **volume quota**. A first-run full bringup
uses ~20 volumes (10 demo VMs + Trilio snapshot ephemerals during
initial backup); re-runs reuse existing resources. `setup_tenant.yml`
Phase 1 prints an **advisory** (non-fatal) with current quota state.
Set quota up-front if tight:

```bash
openstack --insecure quota show <project>           # check current
openstack --insecure quota set --volumes <N> <project>
```

Create any missing resources via Horizon or `openstack` CLI before
running setup plays. If a name differs per cluster (e.g. cluster-specific
target-type naming), update the relevant var in `vars/main.yml`.

### 8. Cert handling

If the new Keystone is on an IP or self-signed hostname, leave the existing
`validate_certs: false` + baked-in `--insecure` flag alone. They're already
on by default in the plays.

If the new cluster has a real public CA cert, you *could* remove them — but
no harm in leaving them. Recommendation: leave alone.

### 9. Validate auth before running anything destructive

```bash
ansible-playbook playbooks/auth_test.yml
```

This prints the `auth_url` actually in use and lists networks visible to the
project. Confirm both match the new cluster before proceeding. If the
`auth_url` looks like a *different* cluster, see [auth-design.md](auth-design.md)
for debugging (most likely cause: stale `~/.config/openstack/clouds.yaml`
entry with cloud name `openstack`).

### 10. Update CLAUDE.md and the Session State

On the new cluster branch:
- Update the **Environment** table at the top of `CLAUDE.md` with the new
  cluster's Keystone, project ID, project name, networks, backup targets.
- Add a one-line note in **Session State** at the bottom: "Cluster: <name>.
  Bringup completed YYYY-MM-DD. All four plays validated against this cluster."
  (Or, until validation: "Bringup in progress, currently at step N.")
- Commit. Push. Tag the commit `bringup-<cluster-slug>` if you like.

### 11. Run the plays

```bash
ansible-playbook playbooks/setup_tenant.yml
ansible-playbook playbooks/setup_trilio.yml
```

Then teardown, then redo. Confirm idempotency on the rebuild.

## What does NOT change per cluster

- Playbook logic, task structure, demo definitions.
- Resource naming convention (`firewall-*`, `webapp-*`, `database-*`).
- Tag scheme (`fw`, `app`, `pg`).
- `demo_prefix` (still defaults empty; only needed if multiple developers
  share a project).
- Auth pattern itself (OS_* env vars from `vars/main.yml` + `vars/vault.yml`,
  no `clouds.yaml`). See [auth-design.md](auth-design.md).
- The `--insecure` flag baked into `openstack_bin` for the workloadmgr CLI.

## History of bringups

| Date | Cluster | Branch | Notes |
|---|---|---|---|
| 2026-04-30 | RHOSO (presales) | `realistic-apps` | First cluster on the OS_* env-var auth pattern. Migrated from RHOSP 17 in same branch. |
| 2026-05-22 | Canonical (TVO) | `canonical` | First multi-clone bringup. New conventions: clone-per-cluster, fresh `uv` venv, **openrc imported into repo** (`vars/openrc/`, gitignored + `.sample`), **`playbooks/discover.yml`** added as reusable prereq enumerator, **volume-quota advisory** in `setup_tenant.yml` Phase 1, `vars/main.yml.sample` rewritten to current OS_* shape. Teardown/rebuild idempotency cycle deferred. |
