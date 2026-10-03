# rustfs

Deploys RustFS (S3-compatible object storage) as a Docker Compose project in `/opt/rustfs`.

## Requirements

- Debian or Ubuntu target.
- The `community.docker` collection on the controller.

## Variables

All settings live in the `rustfs_config` dictionary. Override it as a whole in `host_vars`/`group_vars`: Ansible does not merge dictionaries by default.

| Key | Default | Description |
|---|---|---|
| `version` | `"latest"` | Image tag of `rustfs/rustfs`. |
| `port` | `":9000"` | Host side of the S3 API port mapping, rendered as `<port>:9000` (a leading `:` binds on all interfaces). |
| `console_port` | `":9001"` | Same, for the web console (container port 9001). |
| `data_path` | `"/opt/rustfs/data"` | Host directory mounted as `/data`, created with owner `10001:10001`. |
| `enable_console` | `true` | Enables the web console. |
| `access_key` | `"your_access_key"` | Root access key. Placeholder: always override it with a Vault-encrypted value. |
| `secret_key` | `"your_secret_key"` | Root secret key. Placeholder: always override it with a Vault-encrypted value. |

Newer options are standalone variables rather than keys of that dictionary, precisely because a `host_vars` `rustfs_config` replaces it wholesale: a key added only to the role defaults is undefined on every host that sets the dictionary.

| Variable | Default | Description |
|---|---|---|
| `rustfs_obs_endpoint` | `""` | Base OTLP endpoint RustFS pushes to. Empty leaves the variable unset and RustFS pushes nothing. |
| `rustfs_network_subnet` | `"172.18.0.0/16"` | Subnet of the compose network, pinned rather than left to Docker's address pool. |
| `rustfs_network_gateway` | `"172.18.0.1"` | Gateway of that network - the address a collector on the host is reachable on from the container. |

## Observability

**RustFS exposes no Prometheus endpoint.** Checked on the running binary (1.0.0-beta.8): its only observability option is `--obs-endpoint`, *"Root OTLP endpoint for traces, metrics, and logs"*. Nothing can scrape it, so collecting its metrics means receiving them - an OTLP push into a collector that re-exposes them.

On the storage node that collector is Alloy, already running on the host. The flow:

```
rustfs container  --OTLP/HTTP-->  Alloy on the host  --remote_write-->  Prometheus on core
172.18.0.2                        172.18.0.1:4318
```

Three things have to line up, which is why the network is pinned:

1. `rustfs_obs_endpoint` names the gateway of the compose network.
2. Alloy binds its OTLP receiver to that same gateway address, not `0.0.0.0`, so only this container can reach it (`playbooks/files/alloy-storage.alloy`).
3. The push crosses from the container to the host, so it lands in the host firewall's `input` chain with the container's address as source - the node needs a rule allowing that port from `rustfs_network_subnet` (`host_vars/stor-rpi5-01.phorge.yml`).

With that in place the node reports 165 distinct `rustfs_*` metric names under `job="rustfs"`, covering cluster capacity, drive counts, erasure set health and per-bucket usage.

## Note on the root keys

`rustfs server --help` prints the *values* of the environment variables it reads, including `RUSTFS_SECRET_KEY`. Anyone who can run `docker exec` on the node can read the root S3 credentials without touching the Vault-encrypted inventory. That is a property of the image, not of this role; it is worth knowing when deciding who gets Docker access on the node.

## Dependencies

- `docker`

## Example

See `inventories/production/host_vars/stor-rpi5-01.phorge.yml` for a real configuration (Vault-encrypted keys, data on the RAID array).
