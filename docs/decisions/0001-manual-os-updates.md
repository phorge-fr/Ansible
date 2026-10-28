# 0001: OS updates are manual

Status: accepted.

## Decision

No `unattended-upgrades`, no apt periodic timers, no automatic reboot on any node. The maintainer decides when updates are applied. Patching is an operator-triggered, serial playbook ([patch management](../patch-management.md)).

## Why

Updates on storage and compute nodes interrupt services (NFS, VMs) and must be scheduled; security hot-patches are reacted to deliberately. A role that enabled automatic security updates was written for the SSH hardening work and rejected, then removed.

## Consequences

Any issue that lists "enable automatic security updates" is adjusted, not implemented. If updates need handling, propose an on-demand playbook, not an unattended mechanism.
