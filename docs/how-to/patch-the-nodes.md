---
type: how-to
audience: maintainer patching the nodes in a planned window
verified: 2026-10-03
---

# Patch the nodes

`playbooks/patch.yml` is the manual, interactive patch tool. It sets up no
automatic update mechanism of any kind - patching stays a hand-run action
during a planned maintenance window, so updates can be scheduled and security
hot-patches reacted to deliberately.

## Run it

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
  `armbian-bsp*`, `raspi-firmware`, `u-boot*`, and others). Those are the upgrades
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

## Related

- [How patch management works](../explanation/patch-management-design.md)
- [Decision: OS updates are manual](../decisions/0001-manual-os-updates.md)
