# md0 integrity check on stor-rpi5-01

The md0 RAID5 on `stor-rpi5-01` (4 NVMe behind the ASMedia ASM1184e switch)
holds `/mnt/main`, the NFS export used by Forgejo on the svc cluster and the
rustfs S3 endpoint used by the Longhorn backups. A degraded array surviving a
second dropout takes it down, so its integrity should be verified periodically
- an mdadm `--re-add` only resyncs the blocks changed since the dropout (bitmap
based), it does not prove the re-added disk matches the three others.

## Run a check

The check reads every block, recomputes the parity for each stripe and
compares it with what is stored. It changes no data: where `check` only counts
differences, `repair` would rewrite the parity, and a `repair` should never be
run before the cause of a mismatch is understood.

```bash
echo check > /sys/block/md0/md/sync_action
```

Follow the progress and the result:

```bash
grep -A2 '^md0' /proc/mdstat          # percentage, speed, ETA
cat /sys/block/md0/md/sync_action     # check while running, idle when done
cat /sys/block/md0/md/mismatch_cnt    # must stay at 0
```

A full pass over ~750 GB takes about half an hour at the ~100 MB/s the node
sustains. NFS and rustfs keep working during the run; the concurrent IO is
negligible at that speed.

## Reading the result

- `mismatch_cnt` stays 0 for the whole run: the array is coherent, nothing to
  do until the next check.
- `mismatch_cnt` is non-zero: do NOT run `repair` right away. Identify the
  affected areas (`mdadm -E` on the members, `dd` reads around the reported
  block), check SMART (`nvme-cli`/`smartmontools` are not installed on the
  node yet - see issue #7's out-of-scope note), and only then decide whether
  the parity or the data is the side to trust. Rewriting the parity over
  silently-corrupted data is exactly how a broken block gets replicated
  everywhere.

## History

- 2026-10-03: first full check after the 2026-09-20 `--re-add` of `nvme3` (the
  dropout of 2026-07-06, see issue #7): `mismatch_cnt` stayed 0 for the whole
  run.