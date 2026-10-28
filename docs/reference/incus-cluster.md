---
type: reference
audience: operator who looks up how the Incus cluster is laid out and configured
verified: 2026-09-27
---

# Incus cluster

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
  `host_vars`): the `parent` of Incus's physical `UPLINK` network, that is the
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

## Related

- [Deploy the Incus cluster](../how-to/deploy-the-incus-cluster.md)
- [Incus deployment caveats](../explanation/incus-deployment-caveats.md)
