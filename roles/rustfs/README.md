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

## Dependencies

- `docker`

## Example

See `inventories/production/host_vars/stor-rpi5-01.phorge.yml` for a real configuration (Vault-encrypted keys, data on the RAID array).
