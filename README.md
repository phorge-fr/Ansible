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
| `hpc` (`hpc-gpu`, `hpc-npu`) | `ai-z440-01`, `ai-rpi5-01` | none yet, see the note under Roles | - |
| `compute` | `comp-opti-01` to `03` | `setup-compute.yml` (bootstrap), `configure-compute.yml` (day-2 config), see [docs/incus-installation.md](docs/incus-installation.md) | `lxc.incus.system_settings`, `lxc.incus.ceph`, `lxc.incus.ovn`, `lxc.incus.incus` |
| `all` | every host | `setup-alloy.yml` | `grafana.grafana.alloy` |
| `all` | every host | `setup-hardening.yml` | `ssh_hardening` |
| `all` | every host | `setup-firewall.yml` (one node at a time, `enforce` on `control`, `core`, `svc` and `storage`, `audit` elsewhere) | `firewall` |
| `all` | every host | `setup-timesync.yml` (checks clock drift and fixes it in the same run) | - |
| `all` | every host | `patch.yml` (interactive, one host at a time: asks before upgrading and before rebooting each host) | `patch` |
| `core` | k0s `core` nodes | `setup-longhorn-nodes.yml` (Longhorn node prerequisites: packages and `dm_crypt`) | - |
| `compute` | `comp-opti-01` to `03` | `upgrade-compute.yml` (interactive rolling upgrade of the Incus, Ceph and OVN packages) | `patch` |
| `svc` | `svc-rock64-01` to `03` (Armbian) | `setup-ramlog.yml` | - |
| `all` | hosts with `kernel_cmdline_params` (the Raspberry Pi 5 nodes) | `setup-kernel-cmdline.yml` (appends to the boot command line, reports the hosts that need a reboot, never reboots) | `kernel_cmdline` |

`setup-alloy.yml` reads `alloy_config` from `group_vars`. Only `storage` and `compute` define it today, the other groups fall back to the role default (empty configuration).

The `storage` configuration is [playbooks/files/alloy-storage.alloy](playbooks/files/alloy-storage.alloy): host and container metrics to Prometheus, container and journal logs to Loki, both on `core` over HTTPS with the core CA and basic auth. The credentials are vault values of the `storage` group vars, written to `/etc/alloy-secrets.env` (mode `0600`) and read by the service through the environment.

`setup-ramlog.yml` keeps the Armbian ramlog of the `svc` nodes working next to k3s: the kubelet container logs (`/var/log/pods`) are moved to persistent storage via a symlink so they no longer fill the 50 MB zram nor break the logrotate postrotate hook with stale synced files (hosts without the Armbian ramlog script are skipped, so the playbook can run on any group).

`setup-kernel-cmdline.yml` appends kernel parameters to the boot command line of the Raspberry Pi 5 nodes: idempotent, guarded (single line, `root=` kept, `.bak` backup, read-back), and it reports the hosts that need a reboot instead of rebooting them - OS reboots stay under manual control.

`patch.yml` is the manual, interactive patch tool: it reports the pending apt updates, the packages that rewrite `/boot` and the reboot flag of every targeted host as one plan, asks a single yes/no to apply it (a refusal ends the run without touching anything), then patches the hosts **one at a time** - upgrading each, proving dpkg finished and `/boot` is still bootable, asking whether to reboot it, and waiting for it to come back healthy before touching the next. The first failure stops the rollout. The upgrade runs as a transient systemd unit rather than a child of the SSH session, so a dropped connection or a Ctrl-C can no longer leave dpkg half-way - which is what left `svc-rock64-01` unbootable on 2026-10-03. It sets up no automatic update mechanism of any kind: patching stays a hand-run action during a planned maintenance window. See [docs/patch-management.md](docs/patch-management.md) for the flow and the run commands, and [roles/patch](roles/patch/README.md) for the three entry points it is built from.

`setup-longhorn-nodes.yml` installs what Longhorn checks for on the `core` nodes themselves - `nfs-common` and `cryptsetup`, plus `dm_crypt` through `modules-load.d` - which `RequiredPackages` and `KernelModulesLoaded` report on each `nodes.longhorn.io` object. Without them RWX volumes (the share manager exports them over NFS) and encrypted volumes are unavailable. `rpcbind` comes in as a dependency and is turned off: the share manager serves NFSv4, which needs no portmapper. The two conditions only clear once `longhorn-manager` restarts on each node, since that is when Longhorn evaluates them - see the playbook header.

`upgrade-compute.yml` upgrades the Incus, Ceph and OVN packages on the compute nodes, which `setup-compute.yml` installs once and never touches again. Two phases, because the components do not share the same tolerance for mixed versions: Ceph, OVN and Open vSwitch are rolled **one node at a time** behind a health gate, while the Incus packages are swept across **all members at once** - an Incus member upgraded ahead of the others blocks the cluster until every member agrees on the version, so pausing for a prompt between nodes would keep it blocked for as long as someone takes to answer. `noout` is set around each node's Ceph upgrade and unset in an `always` block, so an aborted run cannot leave Ceph's recovery disabled. The gate checks invariants rather than `HEALTH_OK` - every mon in quorum, every OSD up and in, every PG `active+clean`, every member online, a leader on both OVN raft rings - because the cluster sits on `HEALTH_WARN` for a BlueStore slow-op alert and gating on `HEALTH_OK` would refuse to run at all. `patch.yml` flags these packages in its plan and points here instead of upgrading them in place.

`setup-compute.yml` deploys a 3-node Incus HA cluster (`configure-compute.yml` then applies every later config change: ACME, OIDC, Loki, OpenFGA authorization) (Ceph storage, OVN networking) via the vendored [`lxc.incus`](https://github.com/lxc/incus-deploy) collection - see [docs/incus-installation.md](docs/incus-installation.md) for the variable scheme, the prerequisites that still need real values, and `playbooks/teardown-compute.yml`, its destructive counterpart to return a node to a clean state (`-e teardown_confirm=true`).

## Roles

Project roles, each documented in its own `README.md`:

- [docker](roles/docker/README.md)
- [nvidia_drivers](roles/nvidia_drivers/README.md)
- [nvidia_container_toolkit](roles/nvidia_container_toolkit/README.md)
- [rocm_drivers](roles/rocm_drivers/README.md)
- [rustfs](roles/rustfs/README.md)
- [ssh_hardening](roles/ssh_hardening/README.md)
- [firewall](roles/firewall/README.md)
- [kernel_cmdline](roles/kernel_cmdline/README.md)
- [patch](roles/patch/README.md)

No playbook applies anything to the `hpc` group. `setup-hpc.yml` used to apply `docker`, `rocm_drivers`, `nvidia_drivers` and `nvidia_container_toolkit` to every host of the group at once, which was wrong for `ai-rpi5-01` - a Pi has no discrete GPU and needs neither the ROCm nor the NVIDIA stack. The roles are kept, and so is `hpc_servers` (monitoring and LLM inference stack), ready for a playbook that targets `hpc-gpu` and `hpc-npu` separately.

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

The first two are also enforced by CI: [.github/workflows/validate.yml](.github/workflows/validate.yml) runs `ansible-lint` and `--syntax-check` on every pull request to `main` and on every push to `dev`. It needs no secrets - `vault_pass` is gitignored, and neither check has to decrypt anything - but it does install the pinned collections from `requirements.yml`, since `collections/` is gitignored and a fresh checkout cannot resolve `lxc.incus.*` or `grafana.grafana.*` without them.
