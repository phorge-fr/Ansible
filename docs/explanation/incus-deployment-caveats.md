---
type: explanation
audience: operator who needs to know the limits and past bugs of the Incus bootstrap
verified: 2026-09-27
---

# Incus deployment caveats

What to know about how the cluster is bootstrapped, and the upstream bugs that were worked around. Steps are in [Deploy the Incus cluster](../how-to/deploy-the-incus-cluster.md).

## This is a one-shot bootstrap, not a reconciler

Day-2 changes go through `playbooks/configure-compute.yml`
(`ansible-playbook -i inventories/production/hosts --vault-password-file
vault_pass playbooks/configure-compute.yml`). It compares each key of
`incus_init_shared_config` (cluster-wide, applied once from a single node) and
`incus_init_node_config` (per node, applied on each node) with the live value
and only sets what differs, in dict order - `acme.email` must stay last among
the `acme.*` keys since setting it starts the real Let's Encrypt run. It also
trusts core's CA (rolling, one node at a time) and manages the OpenFGA
authorizer. `setup-compute.yml` is for the initial bootstrap only.

Every task in `lxc.incus.incus` that sets `core.https_address`/
`cluster.https_address`, applies `incus_init.config`, creates networks/storage
pools, bootstraps the cluster and creates join tokens only runs on the
Ansible run where the `incus` package transitions from absent to present.
**Re-running `setup-compute.yml` after the cluster is up does not reapply
changes** to `incus_init` (a new network, a config tweak, a fourth node). This
is the vendored collection's own design. Day-2 changes go through `configure-compute.yml`
(above), not through re-running `setup-compute.yml`.

The `ceph` role's OSD/mon bootstrap tasks are the exception - genuinely
idempotent, safe to re-run.

## Ceph manager modules are broken on Debian 13

Ceph 18.2.7 on Debian 13 runs on Python 3.13 with a PyO3-based `python3-bcrypt`. `ceph-mgr` runs every module in its
own Python sub-interpreter and PyO3 refuses to be imported there (`PyO3 modules do not yet support
subinterpreters`). `mgr_util.py` does `import bcrypt` at module level and every module imports it through
`mgr_module`, so **all** mgr modules fail: that is the permanent `HEALTH_WARN 13 mgr modules have failed
dependencies`, and it is not benign. `balancer`, `pg_autoscaler`, `crash`, `devicehealth`, `rbd_support` and
`prometheus` (so any Ceph metrics) are all down. There is no fixed Debian package.

`bcrypt` is only used by `mgr_util.password_hash()` (dashboard user passwords, unused here) and is the only PyO3
import that happens at import time. `playbooks/tasks/ceph-mgr-bcrypt-shim.yml` drops a stand-in `bcrypt.py` in
`/usr/share/ceph/mgr`, the one directory the mgr puts in front of Debian's real module (it builds `sys.path` as
`mgr_module_path`, then site-packages, then the interpreter path - which is why a `PYTHONPATH` drop-in does not
work), and restarts the local mgr. The mgr only treats directories holding a `module.py` as modules, so the file is
ignored by module discovery, and a package upgrade leaves it alone. `configure-compute.yml` applies it one node at a
time, disables `restful` (a default-enabled, deprecated module that imports `OpenSSL` at module level, which no
stand-in can fix) and then fails loudly if `MGR_MODULE_DEPENDENCY` is still reported; `setup-compute.yml` applies it right after
the roles on a fresh cluster.

After the fix has been rolled out, `ceph -s` may still show `N mgr modules have recently crashed`
(`RECENT_MGR_MODULE_CRASH`): those are crash reports from the broken period (module load failures at the mgr restarts),
posted asynchronously by `ceph-crash` up to ~10 minutes later, so they cannot be archived reliably from the playbook.
Once no new one appears, acknowledge them once with `ceph crash archive-all` (they stay in `ceph crash ls`).
A crash that keeps coming back after that is a real problem.

## The lxc.incus.ceph bug on Ceph 18 with FQDN inventory names

`lxc.incus.ceph` names each mon after `inventory_hostname`
(`comp-opti-01.phorge`) both in `ceph.conf`'s `mon_initial_members` and in the
monmap it generates, but the mon daemon's own identity (`--id`, the systemd
instance, its data directory) uses `inventory_hostname_short`
(`comp-opti-01`). Combined with `ceph_release: distro` skipping
`--set-min-mon-release` (so the generated monmap also lacks proper msgr2/v2
addressing), Ceph 18 mons reject each other during probing (the log says that it is "mostly ignoring" a monitor that is "not part of monmap") and never form quorum.

Pre-seeding a correct monmap (short names, msgr2 addressing) before
`lxc.incus.ceph` runs helps, but isn't sufficient by itself: a mon that
doesn't reach quorum within its ~2 second probe timeout falls back to
reseeding *itself* from `ceph.conf`'s `mon_initial_members` - which the role
still renders with the FQDN names - reintroducing the exact same rejection.
That means `lxc.incus.ceph`'s own "Enable msgr2" handler reliably hard-fails
the play the first time through, with `ceph` client commands hanging for the
full 5-minute `authenticate timed out` before giving up.

`setup-compute.yml` is structured as `tasks: - block: ... rescue: ...` instead
of a plain `roles:` list specifically so this failure can be caught and fixed
rather than stopping the play: the `rescue:` section asserts this is really
the known mon bug (re-raises anything else via `ansible_failed_task`/
`ansible_failed_result`), rewrites `mon_initial_members` to the short names,
rebuilds and injects a corrected monmap, and waits for quorum - which on this
hardware has taken several minutes even with a correct monmap, since the mon
daemons keep electing in the background regardless of how long Ansible
watches (the wait budget is a generous 10 minutes for exactly this reason).
Once quorum holds, it re-runs `lxc.incus.ceph` (idempotent - its bootstrap
tasks are `creates:`-guarded, so this just picks up where the interrupted run
left off), then `lxc.incus.ovn` and `lxc.incus.incus`.

This whole thing is transparent on a normal run - `rescued=1` in the recap
for each host is expected and fine, not a sign anything is actually broken.
It's only worth understanding if a mon ever gets stuck in `state: probing`
outside of this flow (check with `ceph daemon mon.<id> mon_status`).
