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
**Status:** Mid-bringup. Repo not yet cloned to the target path; CLAUDE.md
in the new clone will need to be created from `CLAUDE.md.sample` and
customized.

### Reference material captured

- openrc: `collateral/tvo-canonical-vince-demo-openrc.sh`
- Keystone: `https://172.22.12.33:5000/v3` (self-signed, IP — keep
  `validate_certs: false` + `--insecure`)
- Project: `vince-demo` / `870e66a32c564ed1be66006c1ee5906d`
- User: `vince`
- User domain: `admin_domain` (not `Default`)
- Project domain: `OS_PROJECT_DOMAIN_ID=9291da41e5394e9a89b0ab74e63d6f13`
  — needs `openstack domain show` to resolve to a name
- Region: `RegionOne` (note casing — RHOSO was `regionOne`)

### Checklist

- [x] Decision: clone-per-cluster + fresh `uv` venv per clone
- [x] `docs/new-cluster-bringup.md` written (this file)
- [x] `CLAUDE.md.sample` template added so new clones can `cp` it on day one
- [ ] Branch `canonical` created off `realistic-apps` and pushed
- [ ] Repo cloned into `~/Development/Lab/tvo-canonical/Demos`
- [ ] `CLAUDE.md` in new clone created from `CLAUDE.md.sample` and edited
      for Canonical (Environment section, Session State)
- [ ] Fresh `uv venv` created with ansible-core + openstack.cloud collection
- [ ] `vars/main.yml` created from sample, updated for Canonical auth,
      region, domains
- [ ] `vars/vault.yml` created from sample, populated with Canonical
      password for user `vince`
- [ ] Domain ID `9291da41…` resolved to a domain name (or
      `OS_PROJECT_DOMAIN_ID` plumbed through every play)
- [ ] Tenant network names looked up + written into `vars/main.yml`
- [ ] Prereqs verified in project: keypair `vincent-ansible-key`, SG
      `vbsg-ssh`, boot image, default backup target
- [ ] `playbooks/auth_test.yml` passed against Canonical
- [ ] `setup_tenant.yml` + `setup_trilio.yml` validated end-to-end
- [ ] Teardown + rebuild validated (idempotency)
- [ ] CLAUDE.md Session State updated: bringup complete YYYY-MM-DD
- [ ] Project memory updated to record Canonical bringup completed

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
uv pip install 'ansible-core>=2.16' openstacksdk python-openstackclient workloadmgrclient
ansible-galaxy collection install openstack.cloud
```

Sanity:
```bash
ansible --version
openstack --version
openstack workloadmgr --help | head -5
```

### 3. Auth — find these on the new cluster

Source the new cluster's openrc, or read it. Capture:

| Field | Where it lives | RHOSO example | Canonical example |
|---|---|---|---|
| `OS_AUTH_URL` | openrc | `https://keystone-public-…/` | `https://172.22.12.33:5000/v3` |
| `OS_PROJECT_ID` | openrc / `openstack project show` | `6f030e47…` | `870e66a3…` |
| `OS_PROJECT_NAME` | openrc | `vince-demo` | `vince-demo` |
| `OS_USER_DOMAIN_NAME` | openrc | `Default` | `admin_domain` |
| `OS_PROJECT_DOMAIN_NAME` | openrc / `openstack domain show <id>` | `Default` | resolve from `_ID` |
| `OS_REGION_NAME` | openrc | `regionOne` | `RegionOne` |

**Gotcha:** Some clouds set `OS_PROJECT_DOMAIN_ID` in the openrc instead of
`_NAME`. Resolve to a name with `openstack domain show <id> -f value -c name`,
or add `OS_PROJECT_DOMAIN_ID` to every play's `environment:` block (and drop
`OS_PROJECT_DOMAIN_NAME`).

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
They are not created by these playbooks. Verify and create if missing:

| Resource | Default name | How to verify |
|---|---|---|
| Keypair | `vincent-ansible-key` | `openstack keypair list` |
| Security group | `vbsg-ssh` | `openstack security group list --project $OS_PROJECT_ID` |
| Boot image | (referenced in `setup_tenant.yml` tasks) | `openstack image list` |
| Trilio default backup target | one with `Is_Default=true` | `openstack --insecure workloadmgr backup-target list` |

If the keypair name differs per cluster, change `demo_keypair` in
`vars/main.yml`. Same for `demo_sg`.

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
| 2026-05-22 | Canonical (TVO) | `canonical` | First multi-clone bringup. New conventions: clone-per-cluster, fresh `uv` venv. |
