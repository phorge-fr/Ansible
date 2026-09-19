# nvidia_drivers

Installs the NVIDIA driver package recommended for the host, as reported by the `nvidia-detector` command. If the command is missing or reports nothing, the role does nothing.

## Requirements

- Ubuntu target with the `nvidia-detector` command available.
- The role does not reboot the host; a reboot is required before the driver is loaded.

## Variables

None.

## Dependencies

None.

## Example

```yaml
- hosts: hpc-gpu
  become: true
  roles:
    - nvidia_drivers
```
