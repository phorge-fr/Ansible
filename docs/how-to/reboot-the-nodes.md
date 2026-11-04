---
type: how-to
audience: maintainer rebooting the nodes after a patch window
verified: 2026-11-04
---

# Reboot the nodes

Reboot the hosts that run an older kernel than the one installed, one node at a time, without losing quorum and without interrupting a workload more than the steps below say. Use it after [Patch the nodes](patch-the-nodes.md) lists `REBOOT NEEDED`.

Hosts are interchangeable: the steps name inventory groups and use a placeholder for the node you act on, never a host. The page covers the Kubernetes clusters `core` and `control`, the `compute` Incus cluster, and the `storage` and `hpc` groups. The `svc` cluster is not covered: it was not rebooted when this page was verified, and its boot packages carry a risk that needs their own steps.

## Before you start

- Run every command from the root of this repository, so that `ansible.cfg` supplies the SSH user. The `kubectl` context of the cluster you work on must answer.
- Reboot one node and finish its checks before you touch the next. Never reboot two members of the same cluster at once: etcd, Ceph, OVN and the Incus database each need a majority of their members.
- List the hosts that need a reboot by comparing the running kernel with the newest installed one:

  ```bash
  ansible -i inventories/production/hosts all -b -m shell \
    -a 'echo "$(hostname) running=$(uname -r) newest=$(ls /lib/modules | sort -V | tail -1)"'
  ```

  A host can install several kernel flavours. Compare against the flavour the host runs.

## Reboot one host

The same command reboots every host in the sections below. It returns when the host answers again, or fails after 15 minutes:

```bash
ansible -i inventories/production/hosts <host> -b -m ansible.builtin.reboot \
  -a 'reboot_timeout=900 msg="maintenance reboot"'
```

Then confirm the kernel and the service manager:

```bash
ansible -i inventories/production/hosts <host> -b -m shell \
  -a 'uname -r; systemctl --failed --no-legend | wc -l'
```

The kernel is the new one and the count is `0`.

## Reboot a Kubernetes cluster

Use this for `core` and `control`. Each node is control plane and worker at once. Replace `<cluster>` with the `kubectl` context and the inventory group (`core` or `control`), and `<node>` with the node you reboot. Allow a few minutes per node. On a cluster that runs Longhorn, allow a few more for the volumes to rebuild.

1. Check the cluster before the first node:

   ```bash
   kubectl --context <cluster> get nodes
   kubectl --context <cluster> get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
   kubectl --context <cluster> get kustomizations,helmreleases -A
   ansible -i inventories/production/hosts <cluster> -b -m shell -a 'k0s etcd member-list'
   ```

   All nodes are `Ready`, no pod is listed, every Flux resource is ready, and each controller lists the same etcd members. If the cluster runs Longhorn, also check the volumes:

   ```bash
   kubectl --context <cluster> get volumes.longhorn.io -n longhorn-system \
     -o custom-columns=NAME:.metadata.name,ROBUSTNESS:.status.robustness,NODE:.status.currentNodeID
   ```

   Every volume is `healthy`.

2. Pick the order. Start with a node that is not the holder of the API's keepalived address and that has the fewest attached volumes, and finish with the busiest one. The holder is the node that carries the address set under `keepalived` in the cluster's k0sctl file in Hangar (`ip -br address` on each node). The address moves to another node by itself when its holder reboots.

3. Cordon and drain the node:

   ```bash
   kubectl --context <cluster> cordon <node>
   kubectl --context <cluster> drain <node> --ignore-daemonsets --delete-emptydir-data --timeout=2m
   ```

4. On a cluster that runs Longhorn, the drain stops on `csi-attacher` or `csi-provisioner` in `longhorn-system` with `Cannot evict pod as it would violate the pod's disruption budget`. Delete that one pod, then run the drain again:

   ```bash
   kubectl --context <cluster> -n longhorn-system delete pod <csi-pod>
   ```

   Only these two controllers get this treatment. If another pod blocks the drain, stop and find out why. The Longhorn `instance-manager` budgets release by themselves once the node is cordoned. [Why the CSI pods block](../explanation/rolling-reboots.md#why-a-drain-blocks-on-the-csi-pods) explains why this step is expected.

5. Wait until the evicted pods run elsewhere, and until Flux is ready again. Do not reboot while a pod is still starting:

   ```bash
   kubectl --context <cluster> get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
   kubectl --context <cluster> get kustomizations,helmreleases -A
   kubectl --context <cluster> get pods -A --field-selector spec.nodeName=<node>
   ```

   The first command lists nothing and every Flux resource is ready. The last lists only DaemonSet pods. Pods with a volume take one to two minutes to reattach. The Flux controllers restart on another node, and a resource can report `does not have an artifact` until the source controller is back.

6. Reboot the node with [the command above](#reboot-one-host).

7. Put the node back:

   ```bash
   kubectl --context <cluster> wait --for=condition=Ready node/<node> --timeout=5m
   kubectl --context <cluster> uncordon <node>
   ```

   DaemonSet pods of the node can show `Unknown` for a few seconds while the kubelet reconciles. Wait until every pod is ready.

8. On a cluster that runs Longhorn, wait for the volumes:

   ```bash
   kubectl --context <cluster> get volumes.longhorn.io -n longhorn-system \
     -o custom-columns=NAME:.metadata.name,ROBUSTNESS:.status.robustness
   ```

   Repeat until every volume is `healthy`. They show `degraded` while the replica of the rebooted node rebuilds.

9. Run the checks of step 1 again. Start the next node only when they pass.

## Reboot the compute cluster

The `compute` group is one Incus cluster with Ceph and OVN. An Incus member is evacuated before its reboot, because `cluster.healing_threshold` is 4 seconds: a member that goes away without being evacuated has its instances moved within seconds. Allow a few minutes per member.

Replace `<member>` with the member you reboot. Run the `incus`, `ceph` and `ovn-appctl` commands as root on a member other than that one.

1. Check the cluster before the first member:

   ```bash
   incus cluster list
   incus list --all-projects
   ceph status
   ovn-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound | grep -E '^(Role|Leader)'
   ovn-appctl -t /var/run/ovn/ovnsb_db.ctl cluster/status OVN_Southbound | grep -E '^(Role|Leader)'
   ```

   Every member is `ONLINE`, every Ceph monitor is in quorum, every OSD is `up` and `in`, every placement group is `active+clean`, and both OVN databases name a leader. [`check-compute-health.yml`](../../playbooks/tasks/check-compute-health.yml) tests the same invariants.

2. Evacuate the member and check that it shows `EVACUATED`:

   ```bash
   incus cluster evacuate <member> --force
   incus cluster list
   ```

   The instances of the member move to the other members. The `cluster.evacuate` key of an instance sets how.

3. Reboot the member with [the command above](#reboot-one-host).

4. Wait until the cluster has the member back. On another member:

   ```bash
   ceph status
   ovn-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound | grep -E '^(Role|Leader)'
   ovn-appctl -t /var/run/ovn/ovnsb_db.ctl cluster/status OVN_Southbound | grep -E '^(Role|Leader)'
   ```

   Every monitor is in quorum, every OSD is `up`, and every placement group is `active+clean`. Ceph reports `HEALTH_WARN` for a while. Two warnings are expected and do not block the next member: `1 mgr modules have recently crashed`, and `clock skew detected on mon.<member>`, which clears within a few minutes. Any other warning, or a placement group that is not `active+clean`, does block.

5. Restore the member and check that it is `ONLINE` again:

   ```bash
   incus cluster restore <member> --force
   incus cluster list
   ```

6. Run the checks of step 1 again. Start the next member only when they pass.

## Reboot the storage hosts

The `storage` hosts serve the NFS export that the `svc` cluster uses. The export is unavailable while the host reboots. The NFS clients wait and resume.

1. Check the array before the reboot:

   ```bash
   ansible -i inventories/production/hosts <host> -b -m shell -a 'cat /proc/mdstat'
   ```

   The array `md0` is `active` and shows `[UUUU]`.

2. Reboot the host with [the command above](#reboot-one-host).

3. Check the array, the mount and the export:

   ```bash
   ansible -i inventories/production/hosts <host> -b -m shell \
     -a 'cat /proc/mdstat; findmnt /dev/md0; systemctl is-active nfs-server; exportfs -v'
   kubectl --context svc get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
   ```

   The array shows `[UUUU]` again, it is mounted, `nfs-server` is `active`, the export is listed, and no `svc` pod is in error. If the array is degraded, follow [Check the md0 array](check-the-md0-array.md) before anything else.

## Reboot a host that runs no cluster workload

The `hpc` hosts run no Kubernetes or Incus workload.

1. Check what runs on the host:

   ```bash
   ansible -i inventories/production/hosts <host> -b -m shell -a 'docker ps'
   ```

   The list is empty. If a container runs, stop here and ask its owner.

2. Reboot the host with [the command above](#reboot-one-host).

## Check that it worked

For every rebooted host, `uname -r` reports the newest installed kernel of its flavour and `systemctl --failed` lists nothing. For a cluster, the checks of its first step pass again. Run `patch.yml --check` ([Patch the nodes](patch-the-nodes.md)): the `kernel:` line of a rebooted host no longer starts with `!!`.

## Undo

A reboot cannot be undone. To stop in the middle of a procedure:

- Kubernetes cluster: uncordon the node with `kubectl --context <cluster> uncordon <node>`.
- `compute`: restore the member with `incus cluster restore <member> --force`.

If a node does not come back within the timeout, reboot no other node of the same cluster. The other members keep their quorum and the cluster keeps serving. Leave the node cordoned or evacuated, and reach it through its console.

## Related

- [Why the nodes are rebooted one at a time](../explanation/rolling-reboots.md)
- [Patch the nodes](patch-the-nodes.md)
- [Check the md0 array](check-the-md0-array.md)
