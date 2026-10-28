# Ansible documentation

Pages are grouped by what the reader wants, following [the organization's documentation standard](https://github.com/phorge-fr/.github/blob/main/docs/reference/documentation-standard.md).

## How-to guides

| Page | Task |
| --- | --- |
| [Patch the nodes](how-to/patch-the-nodes.md) | Apply OS updates in a planned window |
| [Deploy the Incus cluster](how-to/deploy-the-incus-cluster.md) | Bootstrap, roll back, or recover the Incus cluster |
| [Change the north-south range](how-to/change-the-north-south-range.md) | Change the address range of a running cluster |
| [Check the md0 array](how-to/check-the-md0-array.md) | Verify the storage node RAID integrity |

## Reference

| Page | Content |
| --- | --- |
| [Incus cluster](reference/incus-cluster.md) | Network layout, address plan, storage, variables, monitoring |

## Explanation

| Page | Question it answers |
| --- | --- |
| [How patch management works](explanation/patch-management-design.md) | Why the patch playbook is built this way |
| [Incus deployment caveats](explanation/incus-deployment-caveats.md) | Limits of the bootstrap and the upstream bugs worked around |
| [The storage node NVMe dropout](explanation/storage-node-nvme-dropout.md) | What happened to the array and what not to do |

## Decisions

| Record | Status |
| --- | --- |
| [0001 OS updates are manual](decisions/0001-manual-os-updates.md) | accepted |
| [0002 The firewall role governs host-terminated ports only](decisions/0002-firewall-scope.md) | accepted |
