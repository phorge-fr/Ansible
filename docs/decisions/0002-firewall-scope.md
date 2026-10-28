---
status: accepted
date: 2026-09-19
decision-makers: l-nmch
---

# The firewall role governs host-terminated ports only

## Context and problem statement

The three Kubernetes clusters run Cilium with `kube-proxy-replacement`, L2 announcements, a VXLAN tunnel, and no Cilium host firewall. With the `firewall` role in `audit` mode, connections from the management network to LoadBalancer addresses and to their NodePorts succeed while the input chain's "unmatched" counters stay at zero. What can the nftables role close?

## Considered options

* Treat nftables as the control for every port on a k0s node.
* Scope the role to host-terminated ports and control the rest elsewhere.

## Decision outcome

Chosen option: "scope the role to host-terminated ports". The eBPF datapath handles LoadBalancer and NodePort traffic before netfilter, so the nftables input chain only governs ports terminated by the host: SSH, the API server, kubelet, etcd, the k0s API, Cilium health, and the node exporter. Docker-published ports on the storage node go through the forward chain instead.

Exposure of LoadBalancer addresses is controlled by the router's default-deny (`phorge-fr/Network`) and, for NodePorts, by `allocateLoadBalancerNodePorts: false` on Services (`phorge-fr/Hangar`).

### Consequences

* Good, because the port lists stay minimal and honest.
* Bad, because the role cannot be presented as closing LoadBalancer or NodePort exposure.

## Pros and cons of the options

### nftables for every port

* Good, because one place.
* Bad, because it does not work: the traffic never reaches the chain.

### Scoped role

* Good, because it matches what the datapath does.
* Bad, because the control is split across three repositories.

## More information

The Cilium host firewall was considered and dropped (Hangar decision 0001): this role already default-denies inbound traffic for hostNetwork pods.
