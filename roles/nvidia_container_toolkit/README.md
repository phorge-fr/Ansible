# nvidia_container_toolkit

Adds the NVIDIA `libnvidia-container` apt repository and installs the NVIDIA Container Toolkit packages, pinned to a fixed version.

## Requirements

- Debian or Ubuntu target.
- Docker installed on the target (see the `docker` role): the role's handler restarts the `docker` service.

## Variables

| Variable | Default | Description |
|---|---|---|
| `nvidia_container_toolkit_version` | `"1.18.0-1"` | Version applied to `nvidia-container-toolkit`, `nvidia-container-toolkit-base`, `libnvidia-container-tools` and `libnvidia-container1`. |

## Dependencies

None declared. Run it after `docker`.

## Example

```yaml
- hosts: hpc-gpu
  become: true
  roles:
    - docker
    - nvidia_container_toolkit
```
