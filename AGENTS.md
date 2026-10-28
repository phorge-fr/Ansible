# AGENTS.md: Ansible

Instructions for anyone working in this repository, human or AI assistant. The organization-wide rules (issues, pull requests, time tracking, security reporting) are in [phorge-fr/.github](https://github.com/phorge-fr/.github/blob/main/AGENTS.md) and [GOVERNANCE.md](https://github.com/phorge-fr/.github/blob/main/GOVERNANCE.md); this file only adds what is specific to Ansible.

## What this repository is

Roles and playbooks for everything that is not Kubernetes: the storage node (NFS and an S3-compatible service), the HPC and GPU nodes, and the Incus compute nodes. The Kubernetes clusters are in `phorge-fr/Hangar`, the router is in `phorge-fr/Network`. The layout and the playbooks are described in the [README](README.md).

## Workflow

- One issue per change, a short branch cut from the current `origin/main`, one pull request with `Closes #N`. Never commit to `main`. Text is in English.
- `main` is protected: `validate` and `governance / pr-governance` must pass.
- Run the checks from the repository root before the pull request:

```bash
ansible-lint
ansible-playbook playbooks/<playbook>.yml --syntax-check
ansible-playbook playbooks/<playbook>.yml --list-hosts
```

`--list-hosts` must list at least one host: a playbook whose `hosts:` pattern matches no group is skipped silently. CI installs the pinned collections of `requirements.yml` first, because `collections/` is gitignored.

## Secrets

Secrets are Ansible Vault values in `inventories/production/{group_vars,host_vars}`: inline `encrypt_string` for values (the diff still shows which key changed), `ansible-vault encrypt` for files copied as-is. The vault password file `vault_pass` is gitignored: never print, copy or commit it. Details in the README.

## Rules specific to this repository

- **Inventory files must match real inventory groups and hosts.** Never add a `group_vars/<x>.yml`, `host_vars/<x>.yml` or an inventory group that does not correspond to something meaningful in `inventories/production/hosts` (no umbrella group invented to share a variable). A small coherent reorganization is fine; propose a new group before creating it. A little duplication across real groups is better than an invented parent.
- **No automatic OS updates.** Never add `unattended-upgrades`, apt periodic timers or automatic reboots. Patching is manual, in a planned window, with `playbooks/patch.yml` ([decision 0001](docs/decisions/0001-manual-os-updates.md)). When a task mentions automatic or security updates, skip that part and say so.
- **The firewall role only governs host-terminated ports.** It cannot close Kubernetes LoadBalancer addresses or NodePorts ([decision 0002](docs/decisions/0002-firewall-scope.md)). Do not promise otherwise.
- Prefer the role over an ad-hoc command on a node. A hand change to a managed host must be reconciled by the role afterwards.
- Before rebooting or touching a node that serves others (the storage node serves NFS to the Forgejo cluster), say what it will interrupt and wait for the maintainer.
- Storage node: never remove, rescan or reset a PCIe device that is a member of the RAID array while another member shares its interrupt line, and never on a degraded array ([runbook](docs/runbooks/stor-nvme-pcie-irq.md)).

## Where to look

[docs/decisions/](docs/decisions/README.md), [docs/runbooks/](docs/runbooks/README.md), [docs/patch-management.md](docs/patch-management.md), [docs/md0-check.md](docs/md0-check.md), [docs/incus-installation.md](docs/incus-installation.md).
