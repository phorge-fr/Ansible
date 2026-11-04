---
type: explanation
audience: maintainer who wants to know why the nodes are rebooted the way the how-to says
verified: 2026-11-04
---

# Why the nodes are rebooted one at a time

The steps are in [Reboot the nodes](../how-to/reboot-the-nodes.md). This page explains the reasons behind them: why one node at a time, why a `core` drain needs a manual step, why an Incus member is evacuated first, and which warnings are expected.

## Context

[`patch.yml`](../how-to/patch-the-nodes.md) installs a new kernel but a running kernel stays the old one until the host restarts. In `fixes` mode the playbook never reboots. In `all` mode it reboots a host only after you answer yes, host by host. A fleet of clustered hosts needs more than that: before a node goes away, the cluster has to be told, or have enough members left to carry on.

The rolling reboot of 2026-11-04 rebooted the three `core` nodes, the three `compute` members, the storage node and the AI node. The steps in the how-to are the ones that run produced.

## Why one node at a time

Every cluster here has three members and needs two of them:

- **`core`**: three etcd members. Two keep the Kubernetes API writable. Each node also runs workloads, so rebooting it also moves its pods.
- **`compute`**: three Ceph monitors, three OSDs with a pool of three replicas and a minimum of two, three OVN database members, three Incus database members. Losing one member of each leaves every service working with reduced redundancy.
- **Longhorn on `core`**: every volume keeps a replica on each of the three nodes. A reboot leaves two healthy replicas, and a second node going away at the same time would leave one.

Rebooting a second member before the first is back and healthy removes the margin that makes the first reboot safe. This is why every section of the how-to ends with a check that must pass before the next node starts.

## Why a drain blocks on the CSI pods

`kubectl drain` evicts pods through the Kubernetes eviction API, which honours PodDisruptionBudgets. Longhorn gives `csi-attacher` and `csi-provisioner` a budget with `minAvailable: 1`. Hangar runs each CSI controller as one replica (`csi.*ReplicaCount: 1` in `base/controllers/longhorn/longhorn.yml`), so the budget allows no disruption and an eviction of the pod on the drained node is refused for ever. The drain waits until its timeout.

Deleting the pod directly does not go through the eviction API, so the budget does not apply. That is safe here for a specific reason: these controllers hold no state, and they came back on another node within about 20 seconds in the 2026-11-04 run. Volume attach and detach operations wait during that gap; every volume attached normally afterwards. It is an exception for these two pods, and the how-to says so. Any other pod that blocks a drain is a signal to stop, not to delete.

The Longhorn `instance-manager` budgets behave differently. They block the first eviction attempt and release by themselves once the node is cordoned and each volume has another healthy replica.

Fixing the cause means running those controllers on more than one replica. That is a Hangar change, tracked in [phorge-fr/Hangar#334](https://github.com/phorge-fr/Hangar/issues/334); when it lands, the special case disappears from the how-to.

Pods that hold a volume and run as a single replica (`authentik-postgresql`, `openfga-postgres`, `loki`, `prometheus`, `alertmanager`) are unavailable while they move, typically one to two minutes. The drain cannot avoid that, and the how-to waits for them before the reboot so a node is never taken down while a pod is half-way to its new home.

## Why an Incus member is evacuated first

The cluster sets `cluster.healing_threshold` to 4 seconds. A member that stops answering for that long is declared offline and its instances are moved to the other members, which is the right reaction to a crash and the wrong one to a planned reboot. Evacuating the member first marks it `EVACUATED` and moves its instances on purpose before it goes away, so the healing has nothing left to move. The restore after the reboot brings the instances back.

## Why no `noout` flag

A common habit is to set the Ceph `noout` flag before rebooting an OSD host, so that Ceph does not mark the OSD `out` and start moving data. It is not used here. An OSD is marked `out` after `mon_osd_down_out_interval`, which is 600 seconds, and a `compute` reboot takes about one minute. With three hosts holding one OSD each and a pool of three replicas, there is also nowhere else to move the data. Leaving the flag out removes one cluster-wide setting that must be unset afterwards, and the how-to does not need to cover the case where it stays set.

## Warnings that are expected

After a `compute` member returns, Ceph reports `HEALTH_WARN`. Two causes are normal:

- **`1 mgr modules have recently crashed`**. A module of the Ceph manager (Ceph 18.2.7) raises an `AttributeError` when the active manager changes. The same error appears in the crash list from September, with no effect on the data. The warning stays until someone archives the crash with `ceph crash archive`.
- **`clock skew detected on mon.<member>`**. The monitor of a freshly booted host is off by more than the 0.05 seconds that Ceph allows until its clock settles. It cleared within a few minutes on `comp-opti-02`, and was still settling on `comp-opti-03` when the member was restored.

Neither is a reason to wait, as long as three monitors are in quorum and every placement group is `active+clean`. [`check-compute-health.yml`](../../playbooks/tasks/check-compute-health.yml) gates on those invariants and on the Incus and OVN state, and it ignores the health string on purpose: this cluster has sat on `HEALTH_WARN` for other reasons before.

## Trade-offs and limits

- The procedure is manual, one command at a time. It trades speed for the chance to look at each step. A playbook that automates it would need the same gates, and none implements them for the clusters yet.
- A reboot is not a rollback point. Once a node runs the new kernel, the only way back is to boot an older one through the bootloader.
- The `svc` and `control` clusters are not covered. The `svc` nodes also have a boot-consistency risk when their kernel and bootloader packages change (see [How patch management works](patch-management-design.md)), which needs its own steps.
- The only `compute` instance during the run was a leftover test VM that was deleted before the last member was evacuated. The movement of real instances is described by the Incus `cluster.evacuate` setting and was not exercised.
