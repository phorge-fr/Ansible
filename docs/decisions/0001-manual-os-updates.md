---
status: accepted
date: 2026-09-19
decision-makers: l-nmch
---

# OS updates are manual

## Context and problem statement

An issue on SSH hardening listed "enable automatic security updates", and a role for it had been written. Updates on storage and compute nodes interrupt services (NFS, VMs). Who decides when updates are applied?

## Decision drivers

* Control over when a node restarts services.
* React deliberately to security hot-patches.

## Considered options

* Automatic updates (`unattended-upgrades`, apt timers, automatic reboot).
* Manual updates through an operator-triggered, serial playbook.

## Decision outcome

Chosen option: "manual updates". No `unattended-upgrades`, no apt periodic timers and no automatic reboot on any node. Patching is `playbooks/patch.yml`, run by hand in a planned window ([how-to](../how-to/patch-the-nodes.md)). The role that enabled automatic updates was removed.

### Consequences

* Good, because nothing restarts a service unannounced.
* Bad, because a node stays unpatched until someone runs the playbook. Any issue that lists automatic updates is adjusted, not implemented.

## Pros and cons of the options

### Automatic updates

* Good, because patched without effort.
* Bad, because uncontrolled restarts of NFS and VMs.

### Manual playbook

* Good, because scheduled, serial, stops at the first failure ([design](../explanation/patch-management-design.md)).
* Bad, because it depends on the maintainer remembering.
