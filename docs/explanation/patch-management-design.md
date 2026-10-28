---
type: explanation
audience: maintainer who needs to know why the patch playbook is built this way
verified: 2026-10-03
---

# How patch management works

`playbooks/patch.yml` is the manual, interactive patch tool. Steps to run it are in [Patch the nodes](../how-to/patch-the-nodes.md).

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

The three bricks live in [roles/patch](../../roles/patch/README.md) - `survey`,
`apt-detached` and `check-boot-consistency` - each with its arguments declared
in the role's `meta/argument_specs.yml`. This playbook keeps only the
orchestration: the plan, the prompts and the serialisation.
