# firewall

Host firewall based on nftables. The role only manages its own table, `inet phorge_filter`, and never touches the tables of Cilium or Docker.

- `input` chain: default deny. Always allowed: loopback, established connections, ICMP essentials (echo requests are rate limited), DHCP client replies and SSH from `firewall_ssh_sources`. Then `firewall_trusted_sources` (everything) and `firewall_allowed_ports`.
- `forward` chain, only when `firewall_published_ports` is not empty: filters the new connections that a DNAT sends to a container (ports published by Docker never go through the `input` chain). Other forwarded traffic is untouched.

## Modes

| `firewall_mode` | Behaviour |
|---|---|
| `audit` (default) | Nothing is blocked. Packets that no rule allows are counted and logged. |
| `enforce` | Packets that no rule allows are dropped. Requires `firewall_ssh_sources`. |

## Variables

| Variable | Default | Description |
|---|---|---|
| `firewall_mode` | `audit` | `audit` or `enforce`. |
| `firewall_ssh_port` | `22` | SSH port. |
| `firewall_ssh_sources` | `[]` | CIDRs or ranges allowed to reach SSH. |
| `firewall_trusted_sources` | `[]` | CIDRs or ranges allowed to reach everything (cluster VLAN, pod CIDR). |
| `firewall_allowed_ports` | `[]` | Services running on the host. |
| `firewall_published_ports` | `[]` | Ports published by Docker. |
| `firewall_log_rate` | `5/minute` | Rate limit of the log rule. |
| `firewall_rollback_seconds` | `180` | Delay of the rollback timer in `enforce` mode. |

Entries of `firewall_allowed_ports` and `firewall_published_ports`:

```yaml
- proto: tcp            # tcp or udp
  ports: [2049]
  sources:              # optional, anyone when omitted
    - 10.3.0.1-10.3.0.3
  comment: NFS from the svc cluster nodes   # no double quotes
```

Shared values live in `group_vars/all.yml`, cluster values in `group_vars/<group>.yml`, and the services of a single machine (NFS, rustfs) in its `host_vars`.

## Rollout

1. Run `playbooks/setup-firewall.yml` in `audit` mode. It goes one node at a time.
2. Let the network live, then look at what would be dropped: `nft list table inet phorge_filter` (counters) and `journalctl -k | grep phorge-firewall`.
3. Add the missing rules to the inventory.
4. Switch one non critical node to `enforce`: `--limit <host> -e firewall_mode=enforce`, then set `firewall_mode: enforce` in its `host_vars` once it is validated.

## Safety net

Before changing anything in `enforce` mode, also with `--check`, the role reads the address the node sees for the current SSH connection. It fails unless that address is covered by `firewall_ssh_sources` and sshd listens on `firewall_ssh_port`. A dry run reports the changes without touching the node, the rules are only loaded in a real run:

```bash
ansible-playbook playbooks/setup-firewall.yml --check --diff --limit <host> -e firewall_mode=enforce
```

Then, in `enforce` mode, the role arms a `systemd-run` timer before loading the rules, then checks from the controller that the SSH port answers on a new connection. If it does, the timer is cancelled. If not, the play fails and the timer deletes the table and disables `phorge-firewall.service`: the node stays reachable, also after a reboot. Running the playbook again restores the firewall.

## Limits

- Tested in a Debian 12 container (audit, enforce, published ports, rollback), not yet on the real nodes: start in `audit`.
- Allow rules match IPv4 sources.
- The check only covers the address Ansible connects from. Every other place you may connect from must be listed in `firewall_ssh_sources`.
- `--check` needs `python3-apt` on the target (any earlier real run of an apt task installs it).
- The `input` chain does not see traffic that Cilium redirects in eBPF. Check the audit counters on the k0s nodes before enforcing.

## Dependencies

None.
