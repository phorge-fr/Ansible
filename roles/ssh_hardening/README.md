# ssh_hardening

Applies an SSH server baseline through a drop-in file, `/etc/ssh/sshd_config.d/00-phorge-hardening.conf`:

- `PermitRootLogin` (see the variable below)
- `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitEmptyPasswords no`
- `PubkeyAuthentication yes`
- `X11Forwarding no`

The role makes sure `sshd_config` includes the drop-in directory, validates every change with `sshd -t` before writing it, reloads `ssh`, then reads the effective configuration with `sshd -T` and fails if it does not match the baseline. `sshd_config` keeps the first value it reads: the `00-` prefix wins over other drop-ins, but a directive placed before the `Include` line of the main file would still win, and the role reports it.

## Requirements

- Debian or Ubuntu target running OpenSSH.
- Key-based access for the account Ansible uses. The role refuses to disable password authentication when `ansible_password` is set.

## Variables

| Variable | Default | Description |
|---|---|---|
| `ssh_hardening_permit_root_login` | `"prohibit-password"` | Value of `PermitRootLogin`. |
| `ssh_hardening_password_authentication` | `false` | Value of `PasswordAuthentication`. |
| `ssh_hardening_password_login_users` | `[]` | Users allowed to log in with a password whatever the global setting (emergency access), through a `Match User` block. |
| `ssh_hardening_password_login_sources` | `[]` | Addresses these users may log in from, as CIDRs or wildcard patterns (sshd does not accept ranges). Anywhere when empty. |

In this inventory (`group_vars/all.yml`) `user` may log in with a password from the management network only. Root stays key-only. Keep a strong password on that account: on several nodes it can use `sudo` or belongs to the `docker` group, which is equivalent to root. The role checks that the exception is effective with `sshd -T -C`.

`k0sctl` connects to the cluster nodes as `root` (see `k0s-cluster.yml` in FrontPlane), so keep `prohibit-password` until it is switched to a sudo user. Only then set the variable to `"no"`.

## Dependencies

None.

## Drift audit

```bash
ansible-playbook playbooks/setup-hardening.yml --check --diff
```

## Example

```yaml
- hosts: all
  become: true
  roles:
    - ssh_hardening
```
