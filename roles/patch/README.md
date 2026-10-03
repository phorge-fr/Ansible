# patch

The pieces `playbooks/patch.yml` is built from. The role holds no `main.yml`
on purpose: it is three independent entry points, included with `tasks_from`.
The playbook keeps the orchestration (the plan, the prompts, `serial: 1`), the
role keeps everything that has to be correct on the host.

| Entry point | What it does |
|---|---|
| `survey` | Heals a half-finished dpkg transaction, then collects the pending packages, the free space and the reboot flag, and builds the host's plan entry. |
| `apt-detached` | Runs one apt/dpkg command as a transient systemd unit and follows it from the outside. |
| `check-boot-consistency` | Asserts that `/boot` can still boot. Read-only. |

```yaml
- ansible.builtin.include_role:
    name: patch
    tasks_from: apt-detached
  vars:
    patch_detached_name: upgrade
    patch_detached_command: apt-get -y upgrade
```

Arguments are declared in `meta/argument_specs.yml`, so a missing one fails
with a clear message instead of building a command from an empty variable.

## Why this exists

On 2026-10-03 an interrupted upgrade left `svc-rock64-01` unbootable and it
had to be recovered by pulling its SD card. The upgrade was a child of the
Ansible SSH session; the session went away while dpkg was configuring
`linux-image`, and dpkg never finished. The card held 50 packages
`install ok unpacked`, a 0-byte `initrd.img-*.new`, and a `/boot/Image`
symlink pointing at the `vmlinuz` the upgrade had already removed - so U-Boot
had no kernel to load. Nothing in apt or dpkg reports that state: the host
keeps running on the kernel already in memory until someone reboots it.

`apt-detached` removes the first link in that chain, `check-boot-consistency`
the last.

## `apt-detached`

The command is staged as `/usr/local/sbin/phorge-patch-<name>`, then started
with `systemd-run --no-block`. Being owned by PID 1, it survives a dropped
connection, an Ansible timeout and a Ctrl-C: only the *watching* dies, never
dpkg. The wrapper records apt's exit status in
`/var/log/phorge-patch-<name>.status` as its last action, and that file
appearing is the completion marker.

An in-progress run is **resumed, not restarted**: the unit is checked for
first, and while one is live neither the script nor the unit is touched.
Overwriting the script under a running bash is how a resumed run would corrupt
its own execution.

Two details are load-bearing rather than cosmetic:

- **`--no-block`**, because a `Type=oneshot` start job is only complete when
  the process exits - without it `systemd-run` blocks for the whole upgrade
  and the bounded wait can never apply.
- **the default `Type=simple`**, because `oneshot` reports the unit as
  `activating` for its entire run, which `systemctl is-active --quiet`
  rejects, so the resume check would miss a live run and try to start a
  second one over it.

Returns `patch_detached.rc`: apt's own status, `124` when the wait gave up
(the unit is still running, deliberately), `125` when nothing could be
started.

## `check-boot-consistency`

| Check | What it catches |
|---|---|
| no dangling symlink in `/boot` | `Image` → a `vmlinuz` the upgrade removed |
| no `*.new` / `*.dpkg-new` / `*.dpkg-tmp` left | a half-written initramfs |
| no empty `vmlinuz*`/`Image*`/`initrd.img*`/`uInitrd*`/`System.map*` | the 0-byte initrd |
| the newest installed kernel has a non-empty initramfs | a kernel shipped with no initrd at all |

The first and third each catch `svc-rock64-01` on their own. Bootloader
agnostic on purpose: the invariants hold on Armbian/U-Boot (`Image`,
`uInitrd`), on the Raspberry Pi and on GRUB alike. Empty files are only
flagged when they match a kernel or initramfs name, because Armbian
legitimately keeps an empty `/boot/.next`.

Sets `patch_boot_problems`, a list of strings, empty when `/boot` looks sane.
The caller decides what to do with it - `playbooks/patch.yml` refuses to offer
a reboot when it is not empty.

## Variables

| Variable | Default | Description |
|---|---|---|
| `patch_detached_timeout` | `3600` | Seconds to follow a detached run before leaving it to systemd. |
| `patch_min_boot_free_mb` | `80` | Free space in `/boot` below which a kernel upgrade is flagged in the plan. |
| `patch_boot_packages_regex` | `linux-image`, `initramfs-tools`, `armbian-bsp`, `u-boot`, ... | Matches the pending packages whose postinst rewrites `/boot`. |
| `patch_detached_name` | - | Required by `apt-detached`: unit, script and log suffix. |
| `patch_detached_command` | - | Required by `apt-detached`: the command to run. |

## Limits

- Debian/apt only, and systemd is required - `apt-detached` is built on
  transient units.
- `check-boot-consistency` checks consistency, not correctness: it cannot tell
  that a present, non-empty initramfs actually boots.
- Neither entry point runs under `--check`: the shell tasks are skipped like
  any other, so a preview can never start a real upgrade.
- Recovering a host that is *already* unbootable is out of scope - that needs
  the boot media and a chroot.

## Dependencies

None.
