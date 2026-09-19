# rocm_drivers

Adds the AMD ROCm apt repository (with a pin priority of 600 for `repo.radeon.com`) and installs the ROCm stack: `rocm`, `rocm-dev`, `rocm-libs`, `rocm-hip-sdk` and `rocblas`.

## Requirements

- Ubuntu target on `amd64` (the repository line is hard-coded to `arch=amd64`, ROCm is not published for other architectures).

## Variables

| Variable | Default | Description |
|---|---|---|
| `rocm_drivers_version` | `"7.1"` | ROCm release, used in the repository URL. |
| `rocm_drivers_ubuntu_codename` | `"noble"` | Ubuntu suite of the repository. Must match the target distribution. |

## Dependencies

None.

## Example

```yaml
- hosts: hpc-gpu
  become: true
  roles:
    - role: rocm_drivers
      vars:
        rocm_drivers_version: "7.1"
```
