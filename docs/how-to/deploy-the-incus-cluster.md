---
type: how-to
audience: operator deploying or rolling back the Incus cluster
verified: 2026-09-27
---

# Deploy the Incus cluster

Runs `playbooks/setup-compute.yml`. How the cluster is laid out: [Incus cluster](../reference/incus-cluster.md).

## Before you start

1. **North-south NIC**: cabled on `comp-opti-01` and `-02`; `comp-opti-03` had no
   carrier on 2026-09-27 - see above.
2. **Cloudflare API token** - done. Scoped token (the DNS edit permission on
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

## Roll out

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

## Roll back

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

## Recover from a failure that is not covered

The mon quorum bug above is the one known, expected, self-healing failure.
If `setup-compute.yml` fails for some *other* reason partway through -
especially after networks/storage pools have already been created - do
**not** just purge the `incus` package and re-run:

- A host that fails mid-play drops out of the batch for the rest of the run,
  and the check of `lxc.incus` that asks whether this host is the bootstrap node
  (`incus_servers[0] == inventory_hostname`) is recomputed from whichever
  hosts are *still active* - so a later step (for example storage pool creation) can
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
`HA_Chassis_Group` and `NAT` row (`ovn-nbctl --bare --columns=_uuid list <table>` to find them, then the matching `ovn-nbctl <table>-del` or `destroy` command) before purging and re-running.

## Related

- [Change the north-south range on a live cluster](change-the-north-south-range.md)
- [Incus deployment caveats](../explanation/incus-deployment-caveats.md)
