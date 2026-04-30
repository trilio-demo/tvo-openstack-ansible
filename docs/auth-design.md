# Auth Design

How these playbooks authenticate to OpenStack, and why.

## Current pattern

Every play sets `OS_*` env vars in its `environment:` block, sourced from
`vars/main.yml` (non-secret) and `vars/vault.yml` (password). Tasks pass
nothing auth-related — `openstack.cloud.*` modules and `openstack` CLI
both pick auth up from the inherited environment.

```yaml
vars_files:
  - ../vars/main.yml
  - ../vars/vault.yml
environment:
  OS_AUTH_URL: "{{ os_auth_url }}"
  OS_USERNAME: "{{ os_username }}"
  OS_PASSWORD: "{{ vault_os_password }}"
  OS_PROJECT_ID: "{{ os_project_id }}"
  OS_PROJECT_NAME: "{{ os_project_name }}"
  OS_USER_DOMAIN_NAME: "{{ os_user_domain_name }}"
  OS_PROJECT_DOMAIN_NAME: "{{ os_project_domain_name }}"
  OS_REGION_NAME: "{{ os_region_name }}"
  OS_INTERFACE: "{{ os_interface }}"
  OS_IDENTITY_API_VERSION: "3"
module_defaults:
  group/openstack.cloud.openstack:
    validate_certs: false
```

No `clouds.yaml` / `secure.yaml` is read by the playbooks. The project still
keeps a `clouds.yaml` + `secure.yaml` at the root for ad-hoc CLI use, but
they're not on the playbook auth path.

## Why not `clouds.yaml` + `secure.yaml`

We tried. Three problems converged:

### 1. Cloud-name collision with the previous environment

`~/.config/openstack/clouds.yaml` already had a cloud entry named `openstack`
pointing at the old RHOSP 17 cluster (`http://172.22.5.22:5000`, user
`vincent`). The project's `clouds.yaml` also used the cloud name `openstack`,
this time for RHOSO. When the SDK looked up `cloud='openstack'`, the search
order picked one silently — and **without** `OS_CLIENT_CONFIG_FILE`, it
picked the user-home one. Calls "succeeded" against the wrong cloud and
returned plausible-looking data, masking the bug.

**Lesson:** never use the generic name `openstack` for a cloud entry when
multiple clouds are likely to coexist on the same machine. Use specific
names like `rhoso` or `osp17`.

### 2. `OS_CLIENT_CONFIG_FILE` does not co-discover `secure.yaml`

Setting `OS_CLIENT_CONFIG_FILE=/path/to/clouds.yaml` makes the SDK load that
clouds.yaml — but **does not** make it look for a sibling `secure.yaml` next
to it. `secure.yaml` discovery still uses the SDK's standard search path
(cwd, `~/Library/Application Support/openstack/`, `~/.config/openstack/`,
`/etc/openstack/`). The project's `secure.yaml` at the repo root is not on
that list.

Empirically, with `OS_CLIENT_CONFIG_FILE` set to the project clouds.yaml:

- `loader._config_files` includes the project file ✓
- `loader._secure_files` does **not** include the project secure.yaml ✗

The SDK then either silently loads an empty/stale `secure.yaml` from
`~/.config/openstack/` or falls back to no password — Keystone returns
401 Unauthorized.

### 3. AnsiballZ subprocess `cwd` is not the controller's `cwd`

`ansible.builtin.command` tasks inherit `cwd = playbook_dir` (i.e.,
`./playbooks/`), not the directory ansible-playbook was invoked from.
`openstack.cloud.*` modules run as AnsiballZ-wrapped subprocesses with
`cwd = some_temp_dir`. Either way, you cannot rely on cwd-relative file
discovery to find project-root files.

This is why a simple "put `clouds.yaml` and `secure.yaml` in the project
root and run from there" approach worked for direct shell calls but
silently broke under Ansible.

## Alternatives we considered

| Option | Verdict |
|---|---|
| Symlink project `clouds.yaml`/`secure.yaml` into `~/Library/Application Support/openstack/` so the SDK finds them on its standard search path. | Workable, but requires per-laptop setup and adds a hidden dependency. |
| Inline password into `clouds.yaml` (single-file auth via `OS_CLIENT_CONFIG_FILE`). | Collapses to one file but mixes secrets and config. |
| Pass `auth:` dict explicitly on every task. | Verbose; same env-var values, just per-task. |
| **OS_* env vars from vault** (current). | Self-contained, no file-search games, matches openrc convention. |

## Debugging pattern: verify `auth_url` early

Authentication succeeding tells you nothing about *which cloud* you hit.
When debugging auth issues, the first move is to dump the URL the SDK
actually used:

```python
import openstack.config
loader = openstack.config.OpenStackConfig()
cc = loader.get_one(cloud='openstack')   # or omit cloud= for env-based
print(cc.get_auth_args()['auth_url'])
print('config_files_loaded:', [f for f in loader._config_files if __import__('os').path.exists(f)])
print('secure_files_loaded:', [f for f in loader._secure_files if __import__('os').path.exists(f)])
```

Run that under both shell and `ansible.builtin.command` to compare. If they
disagree, you have a discovery-path mismatch (cwd, `OS_CLIENT_CONFIG_FILE`,
or stale user-home config).

`playbooks/auth_test.yml` keeps a minimal version of this check that lists
networks and prints the in-use `auth_url`. Run it first whenever auth feels
off.
