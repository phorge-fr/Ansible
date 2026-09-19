# Ansible

![Phorge logo](https://avatars.githubusercontent.com/u/187407936?s=200&v=4)

This repository stores all the ansible configurations & file required to run the [Phorge Cloud Infrastructure](https://phorge.fr).

It covers everything that is not Kubernetes: the storage node, the HPC nodes and the Incus compute nodes. The Kubernetes clusters themselves live in the FrontPlane repository.

## Repository layout

```text
ansible.cfg               # inventory, roles_path and collections_path
.ansible-lint             # lint config (vendored content is excluded)
requirements.yml          # pinned Galaxy roles and collections
inventories/production/
  hosts                   # inventory groups
  group_vars/             # per-group variables (Vault-encrypted secrets inline)
  host_vars/              # per-host variables
playbooks/                # one playbook per concern, see below
roles/                    # project roles (Galaxy roles are installed here but gitignored)
docs/                     # manual procedures not yet automated
```

## Inventory and playbooks

| Group | Hosts | Playbook | Roles |
|---|---|---|---|
| `control`, `core`, `svc` | k0s cluster nodes (managed by FrontPlane) | `setup-alloy.yml` only | `grafana.grafana.alloy` |
| `storage` | `stor-rpi5-01` | `setup-storage.yml` | `rolehippie.mdadm`, `geerlingguy.nfs`, `docker`, `rustfs` |
| `hpc` (`hpc-gpu`, `hpc-npu`) | `ai-z440-01`, `ai-rpi5-01` | `setup-hpc.yml` | `docker`, `rocm_drivers`, `nvidia_drivers`, `nvidia_container_toolkit` |
| `compute` | `comp-opti-01` to `03` | none, see [docs/incus-installation.md](docs/incus-installation.md) | none |
| `all` | every host | `setup-alloy.yml` | `grafana.grafana.alloy` |

`setup-alloy.yml` reads `alloy_config` from `group_vars`. Only `storage` and `compute` define it today, the other groups fall back to the role default (empty configuration).

## Roles

Project roles, each documented in its own `README.md`:

- [docker](roles/docker/README.md)
- [nvidia_drivers](roles/nvidia_drivers/README.md)
- [nvidia_container_toolkit](roles/nvidia_container_toolkit/README.md)
- [rocm_drivers](roles/rocm_drivers/README.md)
- [rustfs](roles/rustfs/README.md)

`hpc-servers` (monitoring and LLM inference stack for the HPC nodes) is not wired to any playbook yet.

External roles and collections are pinned in `requirements.yml`. To upgrade one, bump its version there and run `ansible-galaxy install -r requirements.yml --force`.

Role conventions: names in `snake_case`, every variable prefixed with the role name, fully qualified module names (`ansible.builtin.*`), every task named.

## Setup

requirements:

- [Ansible](https://ansible.com)

1. Install the requirements

```bash
ansible-galaxy install -r requirements.yml
```

2. Test environment

```bash
ansible -m ping all
```

## Encryption and execution

`vault_pass` is the local Vault password file. It is gitignored and must never be committed.

### Secrets conventions

- Values in `group_vars`/`host_vars`: encrypt each secret inline with `encrypt_string` (`!vault` tag), so the diff still shows which key changed.
- Files copied as-is to a host (for example an `.env` file): encrypt the whole file with `ansible-vault encrypt`. `copy` and `template` decrypt it on the fly.

### Encrypt variables

```bash
ansible-vault encrypt_string --vault-password-file vault_pass '<yaml_value_to_encrypt>' --name '<yaml_key>'
```

### Encrypt files

```bash
ansible-vault encrypt_string \
  --vault-password-file vault_pass \
  --stdin-name '<yaml_key>' \
  < file_to.encrypt
```

### Run playbooks with encrypted variables

```bash
ansible-playbook -i inventories/production/hosts playbooks/<playbook>.yml --vault-password-file vault_pass
```

## Quality checks

Run these from the repository root before committing:

```bash
ansible-lint
ansible-playbook playbooks/<playbook>.yml --syntax-check
ansible-playbook playbooks/<playbook>.yml --list-hosts
```

`--list-hosts` must list at least one host: a playbook whose `hosts:` pattern matches no inventory group is skipped silently.
