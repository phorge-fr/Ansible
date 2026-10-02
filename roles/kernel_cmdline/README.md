# kernel_cmdline

Appends kernel command line parameters to the boot file of the Raspberry Pi
nodes: `/boot/firmware/cmdline.txt`, the single line the firmware really boots
from (`/boot/cmdline.txt` is not used).

Hosts without `kernel_cmdline_params`, or without the boot file, are left
untouched, so the role can be applied to the whole inventory.

A broken command line means a node that cannot be fixed remotely, so the role
is heavily guarded:

- it refuses to edit a file that is not a single command line keeping its
  `root=` entry;
- it validates the computed line before writing it (it must be the current one
  extended, nothing removed);
- `copy` keeps a `.bak` backup of the previous file (the boot partition is
  FAT, which forbids colons, so copy's timestamped backup cannot be used);
- the written file is read back and compared to what was computed.

The role never reboots: hosts whose running kernel command line (`/proc/cmdline`)
does not yet contain the parameters are reported at the end of the run.

## Variables

| Variable | Default | Description |
|---|---|---|
| `kernel_cmdline_file` | `/boot/firmware/cmdline.txt` | Boot command line file to edit |
| `kernel_cmdline_params` | `[]` | Parameters to append, set per host in `host_vars` |