---
type: how-to
audience: operator changing the north-south address range of a running Incus cluster
verified: 2026-09-27
---

# Change the north-south range on a live cluster

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

## Related

- [Incus cluster](../reference/incus-cluster.md#address-plan-forwards-and-load-balancers)
