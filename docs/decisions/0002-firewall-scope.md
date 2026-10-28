# 0002: The firewall role governs host-terminated ports only

Status: accepted.

## Context

The three Kubernetes clusters run Cilium with `kube-proxy-replacement`, L2 announcements, a VXLAN tunnel and no Cilium host firewall. Measured with the `firewall` role in `audit` mode: connections from the management network to LoadBalancer addresses and to their NodePorts succeed while the input chain's "unmatched" counters stay at zero.

## Decision

The eBPF datapath handles that traffic before netfilter, so the nftables input chain only governs ports terminated by the host: SSH, the API server, kubelet, etcd, the k0s API, Cilium health and the node exporter. Docker-published ports on the storage node go through the forward chain instead. Exposure of LoadBalancer addresses is controlled by the router's default-deny (`phorge-fr/Network`) and, for NodePorts, by `allocateLoadBalancerNodePorts: false` on Services (`phorge-fr/Hangar`).

## Consequences

When designing minimal port lists for k0s nodes, only host-terminated ports matter. The Cilium host firewall was considered and dropped (see Hangar decision 0001): this role already default-denies inbound traffic for hostNetwork pods.
