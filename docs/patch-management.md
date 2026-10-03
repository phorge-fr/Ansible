# Patch management

`playbooks/patch.yml` is the manual, interactive patch tool. It sets up no
automatic update mechanism of any kind - patching stays a hand-run action
during a planned maintenance window, so updates can be scheduled and security
hot-patches reacted to deliberately.

## The flow

1. **Heal, then survey** every targeted host: finish any dpkg transaction a
   previous run left half-done, refresh the apt cache, list the upgradable
   packages, read the free space on `/boot` and `/`, read the
   `/var/run/reboot-required` flag.
2. **Plan**: one consolidated report - the pending packages per host, which
   upgrades rewrite `/boot`, and any host that looks unsafe to patch. Lines
   that matter are prefixed `!!`.
3. **One decision**: a single yes/no. A refusal ends the run there without
   touching anything.
4. **Then one host at a time** (`serial: 1`, `any_errors_fatal`): upgrade it,
   prove dpkg configured every package and that `/boot` is still bootable,
   ask whether to reboot it, and wait for it to come back healthy before
   moving to the next host. The first failure stops the rollout.
5. **Summary**: per host - packages upgraded, still pending (held or phased
   updates stay pending legitimately), reboot required, rebooted, and the
   kernel it came back on.

## Why it is built this way

On 2026-10-03 an interrupted upgrade left `svc-rock64-01` unbootable. The node
had to be recovered by pulling its SD card and finishing the transaction in an
arm64 chroot. What was found on the card:

- 50 packages `install ok unpacked` and `linux-image-current-rockchip64`
  caught going `half-configured`, with dpkg's journal (`/var/lib/dpkg/updates/`)
  still in place - a transaction interrupted mid-configure;
- `/boot/initrd.img-6.18.54-current-rockchip64.new` at **0 bytes**, and no
  completed `initrd.img`/`uInitrd` for that kernel;
- `/boot/Image` still pointing at `vmlinuz-6.18.35-current-rockchip64`, which
  the upgrade had already removed.

U-Boot therefore had no kernel to load, and the node never came back on the
network. Nothing in apt or dpkg reports that state: the host keeps running
happily on the kernel already in memory until someone reboots it. Three
properties of this playbook each address one link in that chain.

### The upgrade is detached from the SSH session

The `patch` role's `apt-detached` entry point runs apt as a transient systemd unit owned
by PID 1, not as a child of the Ansible SSH session, and then follows it from
the outside. Losing the connection, hitting an Ansible timeout or pressing
Ctrl-C now kills only the *watching*, never dpkg: the transaction always
reaches a consistent state.

An interrupted run is **resumed, not restarted** - the task checks for a unit
that is still active before starting a new one, so re-running the playbook
picks the upgrade up where it stopped instead of stacking a second transaction
on top. The command that ran stays readable on the host at
`/usr/local/sbin/phorge-patch-upgrade`, its output at
`/var/log/phorge-patch-upgrade.log`.

### `/boot` is proven consistent before a reboot is offered

The role's `check-boot-consistency` entry point asserts, after the upgrade and
before the reboot question:

| Check | What it catches |
|---|---|
| no dangling symlink in `/boot` | `Image` → a `vmlinuz` the upgrade removed |
| no `*.new` / `*.dpkg-new` / `*.dpkg-tmp` left | a half-written initramfs |
| no empty `vmlinuz*`/`Image*`/`initrd.img*`/`uInitrd*`/`System.map*` | the 0-byte initrd |
| the newest installed kernel has a non-empty initramfs | a kernel with no initrd at all |

Checks 1 and 3 would each have caught `svc-rock64-01` on their own. It is
bootloader-agnostic on purpose: the invariants hold on Armbian/U-Boot
(`Image`, `uInitrd`), on the Raspberry Pi and on GRUB alike.

If `/boot` is inconsistent the run **fails there and the reboot is never
offered** - the host is still perfectly alive on its loaded kernel, and
rebooting it is precisely what would turn that into a brick. Same for a
package left unconfigured after the upgrade.

### One host at a time, and the first failure stops the rollout

`serial: 1` with `any_errors_fatal: true` means a host is upgraded, verified,
rebooted and back in service before the next is touched. On the 3-node etcd
clusters (`control`, `core`, `svc`) this is what keeps a bad upgrade from
taking the quorum down with it - the previous version upgraded every host in
parallel. After a reboot the host must report `systemctl is-system-running` as
`running` (or `starting`) with a consistent `/boot`, otherwise the run stops
before the next node.

The three bricks live in [roles/patch](../roles/patch/README.md) - `survey`,
`apt-detached` and `check-boot-consistency` - each with its arguments declared
in the role's `meta/argument_specs.yml`. This playbook keeps only the
orchestration: the plan, the prompts and the serialisation.

## Running it

```bash
# preview: shows the plan, prompts for nothing, applies nothing
ANSIBLE_CALLBACK_RESULT_FORMAT=yaml ansible-playbook -i inventories/production/hosts \
  playbooks/patch.yml --vault-password-file vault_pass --check

# interactive run: one yes/no on the plan, then host by host
ANSIBLE_CALLBACK_RESULT_FORMAT=yaml ansible-playbook -i inventories/production/hosts \
  playbooks/patch.yml --vault-password-file vault_pass
```

Targeting and options:

- `--limit 'svc'` - one group at a time is the recommended way to patch the
  k0s clusters, so a maintenance window covers one cluster
- `--limit 'all:!ai-z440-01*'` - exclude a host
- `-e patch_upgrade=dist` - `apt-get dist-upgrade` instead of `upgrade`, when
  new dependencies are wanted (the default takes no removals and no brand-new
  packages)

`ANSIBLE_CALLBACK_RESULT_FORMAT=yaml` is not required, it only makes the plan
and the summary readable (the default JSON-ish callback flattens them).

## Notes

- `--check` is the safe preview: the pauses are skipped in check mode, so the
  plan is shown and nothing is ever applied or rebooted.
- The plan's reboot column shows the flag *before* the upgrade (it reports the
  previous run's leftovers); the per-host reboot question after the upgrade
  uses the fresh flag.
- A host found with packages left broken by an earlier run is healed with
  `dpkg --configure -a` (also detached) during the survey, and the plan says
  so. If it is *still* broken afterwards, that host is refused rather than
  patched on top of a broken state.
- `!! REWRITES /boot` in the plan means the upgrade includes a package whose
  postinst rebuilds the boot files (`linux-image*`, `initramfs-tools`,
  `armbian-bsp*`, `raspi-firmware`, `u-boot*`, ...). Those are the upgrades
  that can leave a host unbootable, and the ones worth a maintenance window.
- If apt is reported as `STILL RUNNING` (exit 124), it was deliberately left
  alone: the unit is still working under systemd. Re-run the playbook to pick
  it up.
- On the `compute` nodes the plan flags the Incus, Ceph and OVN packages as
  `!! owned by playbooks/upgrade-compute.yml, do not upgrade here`. They are
  not held back - the choice stays yours - but upgrading them through this
  playbook's shape is actively wrong, not merely suboptimal: one node at a
  time with a prompt in between is the worst possible order for Incus, which
  blocks the cluster for as long as its members disagree on the version. Use
  `--limit 'all:!compute'` here and `upgrade-compute.yml` for those.
- Recovering a host that is already unbootable is not something this playbook
  can do - it needs the boot media and a chroot. The `svc-rock64-01` procedure
  was: `dpkg --configure -a`, then `update-initramfs -c -k <version>`, then fix
  the `/boot` symlinks.
