---
type: explanation
audience: maintainer who works on the storage node array
verified: 2026-10-28
---

# The storage node NVMe dropout

The storage node holds a RAID5 of four NVMe drives behind a PCIe switch on a Raspberry Pi 5. The array serves the NFS export used by Forgejo and the S3-compatible service.

## What happened

- A drive dropped ("controller is down", status all ones) on a power-management event. The array then ran degraded for 76 days without anyone noticing, because metrics from the node only reached Prometheus later (the `NodeRAIDDegraded` alert fired once they did).
- It was repaired by removing the member, removing the PCI device, rescanning the bus and re-adding the member (bitmap based, seconds). SMART of all four drives was clean.
- **Side effect:** only two drives get MSI-X; the other two share a legacy interrupt line. The PCIe rescan made the kernel log `nobody cared` for that line and disable it, so both drives were polled: about 640 ms per NFS write plus fsync and about 10 MB/s reads, until a reboot.

## Rules

- Never remove, rescan or reset a PCIe member of the array while another member shares its interrupt, and never on a degraded array.
- Confirm with the maintainer before rebooting: it interrupts Forgejo's NFS. Scale the dependent workload down first (a SQLite WAL on NFS is fragile).
- After a repair, run a `check` on the array and read `mismatch_cnt` (see [md0 integrity check](../how-to/check-the-md0-array.md)).

## Permanent fix

The kernel command line needs `nvme_core.default_ps_max_latency_us=0 pcie_aspm=off pcie_port_pm=off` (one single line: a typo means a node that does not boot). It is applied by the `kernel_cmdline` role (issues #6 and #7), not by editing the node.

## History of the integrity checks

- 2026-10-03: first full check after the 2026-09-20 `--re-add` of `nvme3` (the
  dropout of 2026-07-06, see issue #7): `mismatch_cnt` stayed 0 for the whole
  run.
