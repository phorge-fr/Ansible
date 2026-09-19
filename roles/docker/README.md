# docker

Installs Docker Engine (`docker-ce`, `docker-ce-cli`, `containerd.io`, buildx and compose plugins) from the official Docker apt repository, then enables and starts the service.

## Requirements

- Debian or Ubuntu target. The repository URL is selected from `ansible_facts['distribution']`.
- `ansible_lsb.codename` must be available on the target (i.e. `lsb_release` installed), it is used as the apt suite.

## Variables

None.

## Dependencies

None. This role is pulled in automatically by `rustfs`.

## Example

```yaml
- hosts: storage
  become: true
  roles:
    - docker
```
