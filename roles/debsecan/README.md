# debsecan

Installs [debsecan](https://packages.debian.org/trixie/debsecan), the client of
the Debian security tracker: it lists the CVEs that affect the installed
packages and tells which of them already have a fixed version in the Debian
archive. It reports, it never upgrades anything.

The role installs the scanner and nothing else. Running it and reading the
result is not part of the role.

## What it does

- Records the debconf answer `debsecan/report = false` before installing. The
  package's default is to send daily reports, which makes a non-interactive
  install create `/etc/cron.d/debsecan`, a daily job that mails root. Phorge
  runs nothing on a timer on its hosts ([decision 0001](../../docs/decisions/0001-manual-os-updates.md)).
- Installs the package without its recommended packages (a cron daemon and a
  mail transport agent). The install adds the single `debsecan` package.
- Removes `/etc/cron.d/debsecan` if a hand install left one behind.

## Limits

debsecan matches the installed packages against Debian source package names.
The Raspberry Pi (`+rpt`) and Armbian kernels do not come from the Debian
archive, so the report can miss them or get them wrong on those hosts. Check
those kernels against their own vendor's advisories.

debsecan's suite list stops at `bullseye`, so a run names the suite explicitly
(`debsecan --suite trixie`).

## Variables

| Variable | Default | Description |
|---|---|---|
| `debsecan_package` | `debsecan` | Package that provides the command |

## Usage

```bash
ansible-playbook playbooks/setup-debsecan.yml
```
