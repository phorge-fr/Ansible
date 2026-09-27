# Incus cluster installation

The `compute` group (`comp-opti-01` to `03`) is a 3-node Incus HA cluster. It is
deployed by `playbooks/setup-compute.yml` using the project's own official
deployment tooling, [`lxc.incus`](https://github.com/lxc/incus-deploy) (vendored
in `requirements.yml`, pinned to a commit since upstream has no tagged
releases). This replaces the manual, hand-typed procedure that used to live in
this file.

It covers:

- Software Defined Networking with [OVN](https://www.ovn.org/en/): a physical
  `UPLINK` network on the north-south NIC, and a default OVN network for
  instances.
- Redundant storage with real [Ceph](https://ceph.io/) (not MicroCeph - same
  daemons, no snapd): one OSD per node on the spare NVMe disk.
- Cluster tuning, ACME (Let's Encrypt via Cloudflare DNS-01), OIDC and Loki
  logging via `incus config set` keys from `incus_init_shared_config`, and
  OpenFGA authorization - all applied by `configure-compute.yml` (see
  "Important: this is a one-shot bootstrap" below).

Confirmed working end to end from a completely clean node (2026-09-23), with
`ansible-playbook playbooks/setup-compute.yml` and nothing else: `incus
cluster list` shows all three nodes `ONLINE`/`Fully operational`, `ceph -s`
reports 3 mons in quorum, mgr active, 3 osds up/in, `ovn-sbctl show` lists 3
chassis, and `incus network list`/`incus storage list` show `UPLINK`,
`default` and `remote` all `CREATED`. No manual step on the nodes was needed -
see "Environment fixes this playbook applies automatically" below for
everything it had to work around to get there.

## Network layout

Two NICs per node, a decision already made for this cluster:

- `enp0s31f6` (comp-ew, VLAN 70, `10.10.0.0/24`, already up with static
  addresses `10.10.0.1-3`): Incus's API (`core.https_address`) and cluster
  traffic (`cluster.https_address`), Ceph's public network, and OVN's own
  DB/tunnel traffic. All east-west, node-to-node traffic.
- A USB NIC (comp-ns, VLAN 80, `10.11.0.0/24`, no host addresses assigned on
  the router) whose interface name differs per host
  (`enx00e04c4389a4`/`enx00e04c3f8b2a`/`enx00e04c3f8b4f` - see each host's
  `host_vars`): the `parent` of Incus's physical `UPLINK` network, i.e. the
  external/instance-facing north-south uplink.

**This second NIC is cabled and carries live traffic** (VM to VM across nodes and
VM to Internet work, through the OVN routers' external addresses on the
`UPLINK`). Checked 2026-09-27: `comp-opti-01` and `-02` are up, `comp-opti-03`
was `NO-CARRIER` - check with `ip -br link show enx...`. A node whose uplink has no
link cannot carry the north-south traffic of the OVN routers scheduled on it.

## Address plan, forwards and load balancers

| Range | Where | Use |
|---|---|---|
| `10.10.0.0/24` | comp-ew (VLAN 70), router `10.10.0.254` | East-west: cluster, API, Ceph, OVN, BGP sessions |
| `10.11.0.0/24` | comp-ns (VLAN 80), router `10.11.0.254` | North-south: `UPLINK` `ipv4.ovn.ranges` `10.11.0.10-250`, one address per OVN network for its virtual router's external port |
| `10.12.0.0/24` | routed, not a VLAN | Network forwards and load balancers: `UPLINK` `ipv4.routes` |

`10.12.0.0/24` has no gateway and no interface on the router on purpose. OVN only accepts a forward or load balancer
listen address inside the uplink's `ipv4.routes`; Incus announces each one over BGP as a `/32` with the OVN router's
comp-ns address as next hop, and the MikroTik learns it from `bgp_connections` (Network repo). An address that is not
announced is not answered by anything. The session is the one between the nodes' `core.bgp_address` (comp-ew) and the
router's comp-ew address (`bgp.peers.router` on the `UPLINK`, iBGP, AS `65535`, matching `core.bgp_asn`).

```bash
incus network forward create default 10.12.0.10 target_address=10.99.0.5
incus network forward port add default 10.12.0.10 tcp 443 10.99.0.5 443
```

Exposing one to the Internet is a dst-NAT on the router (or an HAProxy backend) towards that listen address, plus the
forward rule that allows it in the Network repo.

`configure-compute.yml` applies `incus_init.network.*.config` on a running cluster (compared key by key with the live
config, the differing keys set in one command). Changing the `UPLINK`'s gateway and ranges on a live cluster is safe:
each OVN network on it drops its now-invalid router address, detaches and gets a new one. Order when the plan changes:
Network first (`tofu apply`, which needs `prevent_destroy` lifted for the comp-ns address), then `configure-compute.yml`.
comp-ns carries live traffic, so this is not a maintenance-free change: the routers of the OVN networks on the `UPLINK`
(and so the egress of the running instances) are interrupted while they re-attach with their new address, and again if
the MikroTik's comp-ns address changes at another moment. See "Changing the north-south range on a live cluster".

### Changing the north-south range on a live cluster

Two things have to change, the MikroTik's address on VLAN 80 (Network repo) and the `UPLINK`'s gateway and ranges
(Incus), and instances are using the uplink. Do it make-before-break so the router never sits on the wrong side:

1. **Network, additive**: give the router its new address next to the old one, by adding
   `{ interface = "comp-ns", address = "<new>.254/24" }` to `ip_addresses` in `terraform.tfvars` while
   `networks.comp-ns.cidr` still has the old range. Nothing is destroyed, so `prevent_destroy` is not in the way.
2. **Incus**: change `incus_init.network.UPLINK.config` in `group_vars/compute.yml` and run `configure-compute.yml`.
   The OVN networks on the uplink re-attach with an address of the new range; the egress of their instances is
   interrupted for that moment only, the router already answers on both ranges.
3. **Network, cleanup**: switch `networks.comp-ns.cidr` to the new range and remove the extra `ip_addresses` entry (same
   resource key, so it is kept) - the old address is the only thing destroyed, with `prevent_destroy` lifted for that one
   apply, and nothing uses it any more.

Check afterwards: `incus network get <network> volatile.network.ipv4.address` is in the new range for every OVN network
(`default`, `demo`), and an instance still reaches the Internet.

## Storage

Each node has two disks: `/dev/sda` (the OS - untouched) and one spare, fully
unpartitioned NVMe. The NVMe is the Ceph OSD, one per node, addressed by its
`/dev/disk/by-id/nvme-eui...` path (stable across reboots, unlike `/dev/nvme0n1`)
in each host's `ceph_disks`. `ceph_roles: [client, mon, mgr, osd]` on all three
nodes (no `mds` - Incus's ceph driver uses RBD, not CephFS; no `rgw` - S3 is
already served by `rustfs` on `stor-rpi5-01`). No per-node local storage pool:
both disks are already spoken for.

## Variables

Everything lives in `inventories/production/group_vars/compute.yml` (shared:
`incus_name`, `incus_release`, `incus_roles`, `ceph_fsid`, `ceph_release`,
`ceph_roles`, `ceph_network_public`, `ovn_name`, `ovn_roles`,
`incus_init_shared_config`, the `incus_init.network`/`incus_init.storage`
dicts) and each host's `host_vars` (`ceph_disks`, `incus_ip_address`/
`ceph_ip_address`/`ovn_ip_address` bound to the comp-ew NIC's fact,
`incus_uplink_interface`, `incus_init_node_config` for the per-node
`core.bgp_address`/`core.bgp_routerid`/`core.metrics_address` keys). The two
get merged with `incus_init_shared_config | combine(incus_init_node_config)` -
Ansible doesn't deep-merge dicts on its own (the same lesson already documented
in `roles/rustfs/README.md`), so this is done explicitly.

Cluster bootstrap-node selection is implicit and alphabetical: `lxc.incus`
sorts the hosts that share `incus_name` and have `cluster` in `incus_roles`,
and the first one becomes the bootstrap node. With these three hosts, that's
`comp-opti-01.phorge` - nothing to configure, just something to know before
renaming a host or adding a fourth one.

## Prerequisites checklist (things outside this repo's reach)

1. **North-south NIC**: cabled on `comp-opti-01` and `-02`; `comp-opti-03` had no
   carrier on 2026-09-27 - see above.
2. **Cloudflare API token** - done. Scoped token (Zone.DNS Edit on
   `phorge.fr` only, not the Global API Key), vaulted as
   `incus_cloudflare_api_token`, fed to lego as
   `CLOUDFLARE_DNS_API_TOKEN` through `acme.provider.environment`. Setting
   `acme.email` starts a real Let's Encrypt registration and DNS-01 challenge
   immediately, which is why it must stay the last `acme.*` key in
   `incus_init_shared_config` (dict order is the apply order).
3. **OIDC + OpenFGA** - done, kept here for the reasoning behind it.
   OIDC: FrontPlane's `blueprint-incus.yml` registers a public client (PKCE,
   no secret); `oidc.client.id` and `oidc.issuer`
   (`https://auth.phorge.fr/application/o/incus/`) are in
   `incus_init_shared_config`. The web UI callback is strict:
   `https://iaas.phorge.fr/oidc/callback`. CLI login uses the device flow
   (`incus remote add <name> https://10.10.0.1:8443 --auth-type=oidc` prints
   a code to enter on `https://auth.phorge.fr/device`).
   OpenFGA: without an authorizer, Incus lets any authenticated user do
   anything (an OIDC user created an instance with no grant). It is enabled
   by `configure-compute.yml`, not by `setup-compute.yml`: the store has to
   exist first, and the three keys `authorization.openfga.api.url`,
   `.api.token` and `.store.id` (the names on Incus 7.4 - the old
   `openfga.*` ones are "unknown key") must be set together in one command.
   It also grants `admin` on `server:incus` to `incus_openfga_admins`,
   otherwise nobody but a TLS client certificate could do anything.
   Prerequisites, all in place: the Network repo's `incus-comp-to-core` rule
   (nodes to `10.2.0.10:443`), FrontPlane's Ingress without Traefik BasicAuth
   (Basic and Bearer share the `Authorization` header, a client can't send
   both), and core's local root CA in the nodes' system trust store
   (`playbooks/tasks/trust-core-ca.yml` - Incus's OpenFGA client has no CA
   option, unlike the Loki logger's `target.ca_cert`).
4. **Loki `iaas` password** - done. Logs from Incus reach
   `https://loki.core.phorge` through the same `incus-comp-to-core` rule; the
   password is `incus_loki_password` (vaulted), matching the `iaas` entry of
   FrontPlane's `loki-basic-auth-secret`.
5. **BGP peering** - done. The Network repo's `bgp_connections` peers the
   router (`10.10.0.254`) with `10.10.0.1-3` (iBGP, AS 65535, matching
   `core.bgp_asn`); the per-node `core.bgp_address`/`core.bgp_routerid` come
   from `incus_init_node_config`.

Only the north-south NIC of `comp-opti-03` (no carrier when last checked) is still open. Everything else in
`incus_init_shared_config` is applied and can be changed at any time with
`configure-compute.yml`.

## Ceph mgr modules are broken on Debian 13 (and the fix)

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

## Monitoring (Grafana Alloy on every node)

`playbooks/files/alloy-compute.alloy`, installed with `setup-alloy.yml --limit compute`. Every node monitors itself,
so losing a node or an Alloy never blinds the others, and the config deals explicitly with anything that could
otherwise be stored three times:

| What | Source | Why there is no triplet |
|---|---|---|
| Node metrics | `prometheus.exporter.unix` on each node | Local by nature. Virtual interfaces (tap, veth, OVN ports) and Incus/Ceph mounts are excluded so cardinality does not grow with the instances. |
| Incus metrics | each node scrapes its own `core.metrics_address` | Per-node and per-instance series are local to a member. The cluster-wide ones (`incus_project_resources_total`, `incus_operations_total`, `incus_storage_pool_*`) are identical on all nodes and are dropped; `incus_warnings_total` is kept from every node on purpose (use `max without (instance)`). |
| Ceph metrics | each node scrapes its local mgr on `:9283` | Only the active mgr listens on `:9283` (the standbys do not open it): one copy, following the active mgr on failover. The two standby nodes report `up{job="ceph-mgr"} == 0` and log a "connection refused" scrape error each interval, which is expected: alert on the absence of `ceph_health_status` or on `max(up{job="ceph-mgr"}) == 0`, never on a single instance. |
| Journal | each node's own journal (incus, ceph-*, ovn-*, ssh, ...) | Local. Daemon logs are read from the journal only, never also from `/var/log/ceph`. |
| OVN / Open vSwitch logs | `/var/log/ovn`, `/var/log/openvswitch` | Files only, node-local. |
| Ceph cluster and audit logs | `/var/log/ceph/ceph.log`, `ceph.audit.log` | Every mon writes an identical copy. Sent without the `instance` label so the three copies land in one stream with the same timestamps and lines, which Loki ignores as duplicates. |
| Incus native Loki logger | Incus itself, `lifecycle,network-acl` only | Each member sends only its own events (checked in Loki: `instance == location` on every stream), and the daemon logs are not sent a second time. |

All scrapes are local, so enforcing the firewall later needs no monitoring port open between nodes. Prerequisites:
the `compute` user must exist in FrontPlane's Prometheus and Loki basic-auth secrets, then vault
`alloy_prometheus_basic_auth_password` and `alloy_loki_basic_auth_password` in `group_vars/compute.yml`.
`configure-compute.yml` enables the Ceph `prometheus` module and sets `mgr/prometheus/exclude_perf_counters=false`
(nothing here deploys a `ceph-exporter`, without it the OSD/mon counters would be missing).

## Environment fixes this playbook applies automatically

`setup-compute.yml`'s `pre_tasks` fix several things found on these specific
nodes that would otherwise need a manual SSH session before the vendored
roles could even start. All are idempotent - harmless to run again on a node
that's already fine:

- **DNS.** These nodes never had a resolver configured (a gap in their
  original, pre-Ansible static network setup) - `apt` couldn't resolve
  `deb.debian.org` at all. Installs `resolvconf`, adds `dns-nameservers
  <gateway>` to the static interface config, and feeds it immediately so no
  reboot is needed.
- **jmespath on the controller.** `lxc.incus.incus`'s `community.general.
  json_query` calls need the `jmespath` Python library in *your own* Ansible
  environment, not the managed nodes. Installed once via `pip`.
- **Ceph OSD disk signatures.** A disk that was ever used for Ceph before
  (by an earlier partial run of this playbook, or by anything else) keeps a
  filesystem/LVM signature that makes `ceph-volume` refuse to touch it. Wiped
  with `wipefs` (not `ceph-volume lvm zap`: that tool isn't installed yet at
  this point) - but only if there's no stamp file proving *we* already
  bootstrapped an OSD there.
- **`/var/lib/ceph/{mon,osd}` ownership.** These specific nodes had those two
  directories owned by `root:root` since long before this automation existed
  (base OS image or an earlier manual test) - `ceph-mon --mkfs` refuses with
  "Permission denied" until they're owned by the `ceph` system user, and
  installing the Ceph packages doesn't retroactively fix ownership on a
  directory it finds already there.
- **A correct Ceph monmap, pre-seeded before `lxc.incus.ceph` can generate a
  broken one.** See the next section for why the vendored one is broken and
  why this alone isn't quite enough on its own.

## The real lxc.incus.ceph bug on Ceph 18 (reef) with FQDN inventory names

`lxc.incus.ceph` names each mon after `inventory_hostname`
(`comp-opti-01.phorge`) both in `ceph.conf`'s `mon_initial_members` and in the
monmap it generates, but the mon daemon's own identity (`--id`, the systemd
instance, its data directory) uses `inventory_hostname_short`
(`comp-opti-01`). Combined with `ceph_release: distro` skipping
`--set-min-mon-release` (so the generated monmap also lacks proper msgr2/v2
addressing), Ceph 18 mons reject each other during probing ("mostly ignoring
mon.X, not part of monmap") and never form quorum.

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

## Recovering from a genuinely different failure

The mon quorum bug above is the one known, expected, self-healing failure.
If `setup-compute.yml` fails for some *other* reason partway through -
especially after networks/storage pools have already been created - do
**not** just purge the `incus` package and re-run:

- A host that fails mid-play drops out of the batch for the rest of the run,
  and `lxc.incus`'s "am I the bootstrap node" check
  (`incus_servers[0] == inventory_hostname`) is recomputed from whichever
  hosts are *still active* - so a later step (e.g. storage pool creation) can
  silently run on a **different** node than the one that created the
  networks.
- Purging just `incus` and retrying leaves Ceph and OVN state behind: the
  `remote` pool's underlying Ceph RBD pool (`incus_<incus_name>`) and OVN's
  logical switches/router/address-sets/port-groups/DHCP options/HA chassis
  group for the `default` network are NOT tied to Incus's own database, so a
  fresh Incus reuses the same object names (`incus_compute`, `incus-net2-*`)
  and collides with them (`Erreur : object already exists` /
  `Erreur : Pool '...' seems to be in use by another Incus instance`).

The reliable fix is `playbooks/teardown-compute.yml` (it wipes Ceph and OVN
too, not just Incus) followed by a clean `setup-compute.yml` run. If you want
to retry just the Incus half without resetting Ceph/OVN, you have to clean
both by hand first: `ceph osd pool rm incus_<incus_name> incus_<incus_name>
--yes-i-really-really-mean-it` (after `ceph config set mon
mon_allow_pool_delete true`), and on OVN, delete every `Logical_Router`,
`Logical_Switch`, `Address_Set`, `Port_Group`, `DHCP_Options`,
`HA_Chassis_Group` and `NAT` row (`ovn-nbctl --bare --columns=_uuid list
<table>` to find them, then the matching `ovn-nbctl <table>-del`/`destroy`
command) before purging and re-running.

## Important: this is a one-shot bootstrap, not a reconciler

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
**Re-running `setup-compute.yml` after the cluster is up will not reapply
changes** to `incus_init` (a new network, a config tweak, a fourth node). This
is the vendored collection's own design. Day-2 changes go through `configure-compute.yml`
(above), not through re-running `setup-compute.yml`.

The `ceph` role's OSD/mon bootstrap tasks are the exception - genuinely
idempotent, safe to re-run.

## Rollout

1. `ansible-galaxy install -r requirements.yml --force`.
2. `pipx run ansible-lint` and `ansible-playbook playbooks/setup-compute.yml --syntax-check`.
3. `--check --diff` has limited value here - almost every task is
   `ansible.builtin.command` wrapped in `creates:`/`when: ...changed` guards
   rather than a module with real check-mode support.
4. `ansible-playbook playbooks/setup-compute.yml` against the whole `compute`
   group in one go - no need to stage node by node, the play already handles
   the bootstrap-node election and the mon quorum recovery itself. Budget
   15-20 minutes for a fresh cluster (the known mon rescue alone can take
   most of that on this hardware).
5. `ansible-playbook playbooks/configure-compute.yml`: applies the day-2 config (ACME, OIDC, Loki logging, OpenFGA
   authorization), trusts core's CA and fixes the Ceph mgr modules, see the sections above.
6. Confirm: `incus cluster list` shows all three `ONLINE`, `ceph -s` shows 3 mons in quorum, 3 osds up/in and
   **`HEALTH_OK`** (a lingering `N mgr modules have failed dependencies` is the Debian 13 PyO3 problem described
   above: it means the mgr fix did not apply, not something to ignore), `ovn-sbctl show` lists three chassis, `incus
   network list`/`incus storage list` show `UPLINK`, `default`, `remote`.
7. `firewall_mode` for `compute` is left in `audit` by this change - a
   follow-up pass derives the real ports (8443 cluster/API, Ceph mon
   3300/6789 and OSD 6800-7300, OVN 6641/6642 and geneve 6081) from the
   `audit`-mode counters, the same way it was already done for the k0s and
   storage nodes.

## Rollback

`playbooks/teardown-compute.yml` returns a node to a clean state - no Incus,
Ceph or OVN packages, state directories, or the Zabbly apt source, and a
reboot to clear any loaded Open vSwitch/LVM state. It refuses to run without
`-e teardown_confirm=true`, since it destroys the Incus cluster, every
instance, the Ceph pool and every OSD's data:

```bash
ansible-playbook playbooks/teardown-compute.yml --limit <host> -e teardown_confirm=true
```

Confirmed working end to end (2026-09-23) on all three nodes, including the
reboot.
