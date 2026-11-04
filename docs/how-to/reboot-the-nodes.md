---
type: how-to
audience: maintainer rebooting the nodes after a patch window
verified: 2026-11-04
---

# Reboot the nodes

Reboot the hosts that run an older kernel than the one installed, one node at a time, without losing quorum and without interrupting a workload more than the steps below say. Use it after [Patch the nodes](patch-the-nodes.md) lists `REBOOT NEEDED`.

This page covers the `core` cluster, the `compute` Incus cluster, the storage node and the AI node. The `svc` and `control` clusters are not covered: they were not rebooted when this page was verified.

## Before you start

- Run every command from the root of this repository, so that `ansible.cfg` supplies the SSH user. The `kubectl` contexts `core` and `svc` must work.
- Reboot one node and finish its checks before you touch the next. Never reboot two members of the same cluster at once: etcd, Ceph, OVN and the Incus database all need two of three members.
- To list the hosts that need it, compare the running kernel with the newest installed one of the same flavour:

  ```bash
  ansible -i inventories/production/hosts all -b -m shell \
    -a 'echo "$(hostname) running=$(uname -r) newest=$(ls /lib/modules | sort -V | tail -1)"'
  ```

  A Raspberry Pi has two flavours (`rpi-v8` and `rpi-2712`). Compare against the one it runs.

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

## Reboot the core cluster

The three `core-wyse` nodes are control plane and worker at once. Expect about 3 minutes of reboot, then about 5 minutes for Longhorn to rebuild its replicas.

1. Check the cluster before the first node:

   ```bash
   kubectl --context core get nodes
   kubectl --context core get volumes.longhorn.io -n longhorn-system \
     -o custom-columns=NAME:.metadata.name,ROBUSTNESS:.status.robustness,NODE:.status.currentNodeID
   kubectl --context core get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
   ansible -i inventories/production/hosts core-wyse-01.phorge -b -m shell -a 'k0s etcd member-list'
   ```

   All nodes are `Ready`, every volume is `healthy`, no pod is listed, and etcd shows three members.

2. Pick the order. Start with the node that holds the fewest attached volumes and not the keepalived address of the API, and finish with the busiest one. The address moves to another node by itself when its holder reboots.

3. Cordon and drain the node:

   ```bash
   kubectl --context core cordon <node>
   kubectl --context core drain <node> --ignore-daemonsets --delete-emptydir-data --timeout=2m
   ```

4. If the drain stops on `csi-attacher` or `csi-provisioner` in `longhorn-system` with `Cannot evict pod as it would violate the pod's disruption budget`, delete that one pod, then run the drain again:

   ```bash
   kubectl --context core -n longhorn-system delete pod <csi-pod>
   ```

   Only these two controllers get this treatment. If another pod blocks the drain, stop and find out why. The Longhorn `instance-manager` budgets release by themselves once the node is cordoned. [Why the CSI pods block](../explanation/rolling-reboots.md#why-a-drain-blocks-on-the-csi-pods) explains the exception.

5. Wait until the evicted pods run elsewhere. Do not reboot while one is still starting:

   ```bash
   kubectl --context core get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
   kubectl --context core get pods -A --field-selector spec.nodeName=<node>
   ```

   The first command lists nothing. The second lists only DaemonSet pods (`alloy`, `cilium`, `nllb`, `engine-image`, `longhorn-csi-plugin`, `longhorn-manager`, `traefik-public`). The pods with a volume (`authentik-postgresql`, `openfga-postgres`, `loki`, `prometheus`, `alertmanager`) take one to two minutes to reattach.

6. Reboot the node with [the command above](#reboot-one-host).

7. Put the node back and wait for the volumes:

   ```bash
   kubectl --context core wait --for=condition=Ready node/<node> --timeout=5m
   kubectl --context core uncordon <node>
   kubectl --context core get volumes.longhorn.io -n longhorn-system \
     -o custom-columns=NAME:.metadata.name,ROBUSTNESS:.status.robustness
   ```

   Repeat the last command until every volume is `healthy`. The volumes show `degraded` while the replica of the rebooted node rebuilds.

8. Run the checks of step 1 again. Start the next node only when they pass.

## Reboot the compute cluster

The three `comp-opti` members form one Incus cluster with Ceph and OVN. An Incus member is evacuated before its reboot, because `cluster.healing_threshold` is 4 seconds: a member that goes away without being evacuated has its instances moved within seconds. Expect about 1 minute of reboot per member.

Run the `incus`, `ceph` and `ovn-appctl` commands as root on a member other than the one you reboot.

1. Check the cluster before the first member:

   ```bash
   incus cluster list
   incus list --all-projects
   ceph status
   ovn-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound | grep -E '^(Role|Leader)'
   ovn-appctl -t /var/run/ovn/ovnsb_db.ctl cluster/status OVN_Southbound | grep -E '^(Role|Leader)'
   ```

   Every member is `ONLINE`, Ceph has three monitors in quorum, three OSDs `up` and `in`, and every placement group `active+clean`, and both OVN databases name a leader. [`check-compute-health.yml`](../../playbooks/tasks/check-compute-health.yml) tests the same invariants.

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

   Three monitors are in quorum, three OSDs are `up`, and every placement group is `active+clean`. Ceph reports `HEALTH_WARN` for a while. Two warnings are expected and do not block the next member: `1 mgr modules have recently crashed`, and `clock skew detected on mon.<member>`, which clears within a few minutes. Any other warning, or a placement group that is not `active+clean`, does block.

5. Restore the member and check that it is `ONLINE` again:

   ```bash
   incus cluster restore <member> --force
   incus cluster list
   ```

6. Run the checks of step 1 again. Start the next member only when they pass.

## Reboot the storage node

`stor-rpi5-01` serves the NFS export that the `svc` cluster uses. The export is unavailable for about 30 seconds. The NFS clients wait and resume.

1. Check the array before the reboot:

   ```bash
   ansible -i inventories/production/hosts stor-rpi5-01.phorge -b -m shell -a 'cat /proc/mdstat'
   ```

   The array `md0` is `active` and shows `[UUUU]`.

2. Reboot the node with [the command above](#reboot-one-host).

3. Check the array, the mount and the export:

   ```bash
   ansible -i inventories/production/hosts stor-rpi5-01.phorge -b -m shell \
     -a 'cat /proc/mdstat; findmnt /mnt/main; systemctl is-active nfs-server; exportfs -v'
   kubectl --context svc get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
   ```

   The array shows `[UUUU]` again, `/mnt/main` is mounted, `nfs-server` is `active`, the export is listed, and no `svc` pod is in error. If the array is degraded, follow [Check the md0 array](check-the-md0-array.md) before anything else.

## Reboot the AI node

`ai-rpi5-01` runs no cluster workload.

1. Check what runs on it:

   ```bash
   ansible -i inventories/production/hosts ai-rpi5-01.phorge -b -m shell -a 'docker ps'
   ```

   The list is empty. If a container runs, stop here and ask its owner.

2. Reboot the node with [the command above](#reboot-one-host).

## Check that it worked

For every rebooted host, `uname -r` reports the newest installed kernel of its flavour and `systemctl --failed` lists nothing. For a cluster, the checks of its first step pass again. Run `patch.yml --check` ([Patch the nodes](patch-the-nodes.md)): the `kernel:` line of a rebooted host no longer starts with `!!`.

## Undo

A reboot cannot be undone. To stop in the middle of a procedure:

- `core`: uncordon the node with `kubectl --context core uncordon <node>`.
- `compute`: restore the member with `incus cluster restore <member> --force`.

If a node does not come back within the timeout, reboot no other node of the same cluster. The other two members keep their quorum and the cluster keeps serving. Leave the node cordoned or evacuated, and reach it through its console.

## Related

- [Why the nodes are rebooted one at a time](../explanation/rolling-reboots.md)
- [Patch the nodes](patch-the-nodes.md)
- [Check the md0 array](check-the-md0-array.md)
