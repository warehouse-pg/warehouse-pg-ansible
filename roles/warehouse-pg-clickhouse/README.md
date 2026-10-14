# warehouse-pg-clickhouse

Installs `edb-clickhouse-server` and `edb-clickhouse-client` from the EDB repository on the hosts in the `monitoring_stack` inventory group. ClickHouse stores the telemetry that WEM reads and that the OTel gateway collector writes.

The role:

- installs the server and client packages,
- writes the listen addresses, ports and logger settings to `/etc/clickhouse-server/config.d/99-ansible-network.xml` and `99-ansible-logger.xml`,
- creates the `clickhouse_user` user with `clickhouse_password` in `/etc/clickhouse-server/users.d/ansible-users.xml`, and restricts the shipped `default` user (empty password) to loopback connections (`127.0.0.1` and `::1`),
- starts the service with `clickhouse start`.

The shipped `users.xml` and `config.xml` are not modified, ClickHouse merges the `*.d/` override files at startup.

## Run

```bash
ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags warehouse-pg-clickhouse
```

## Where the role runs

The role runs on the `monitoring_stack` group, which holds the host selected by `monitoring_host` (falls back to `wem_host` when `monitoring_host` is not set). The `edb-repository` role runs first on the same host.

The role supports Red Hat based systems only.

## Prerequisites

- *EDB repository* configured on the target host (done by `edb-repository` in the playbook).
- `clickhouse_user` and `clickhouse_password` must be set in the inventory, on the `monitoring_host` entry or in the cluster vars. Pre-flight fails if:
  - `clickhouse_user` is missing, is `default`, or contains characters other than letters, digits and underscore,
  - `clickhouse_password` is missing or shorter than `clickhouse_password_min_length` (11) characters.
- `ip_address` must be set for the host, because the default `clickhouse_listen_hosts` uses it. Pre-flight fails on an empty listen address.

> **Note:** the created user accepts connections from any address (`::/0`). Restrict access with firewall or security group rules.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `clickhouse_user` | _(none - required)_ | ClickHouse user to create. Must not be `default`. |
| `clickhouse_password` | _(none - required)_ | Password for `clickhouse_user`, at least `clickhouse_password_min_length` characters. |
| `clickhouse_password_min_length` | `11` | Minimum password length enforced by pre-flight. |
| `clickhouse_server_package` | `edb-clickhouse-server` | Server package name. |
| `clickhouse_client_package` | `edb-clickhouse-client` | Client package name. |
| `clickhouse_version` | `""` | Pin a specific version (e.g. `24.3.1`). Leave empty to install whatever the EDB repository provides. |
| `clickhouse_upgrade` | `install-but-no-automatic-upgrades` | Install/upgrade/remove policy, see below. |
| `clickhouse_listen_hosts` | `["127.0.0.1", "<ip_address>"]` | Addresses ClickHouse binds to: loopback plus the host's private `ip_address`. Do not use `ansible_host`, on cloud hosts it is the public IP. |
| `clickhouse_http_port` | `8123` | HTTP port. |
| `clickhouse_tcp_port` | `9000` | Native protocol port. |
| `clickhouse_log_level` | `information` | Server log level. |
| `clickhouse_log_dir` | `/var/log/clickhouse-server` | Log directory. |
| `clickhouse_log_file` | `<clickhouse_log_dir>/clickhouse-server.log` | Main log file. |
| `clickhouse_error_log_file` | `<clickhouse_log_dir>/clickhouse-server.err.log` | Error log file. |
| `clickhouse_service_name` | `clickhouse-server` | systemd service name, used for diagnostics when the start fails. |
| `clickhouse_service_started` | `true` | `true` runs `clickhouse start`, `false` runs `clickhouse stop`. |
| `DNF_TIMEOUT` | `120` | Maximum time (seconds) to wait for the RPM repository. |

## Package state

| Value of `clickhouse_upgrade` | Effect |
|---|---|
| `install-but-no-automatic-upgrades` (default) | Install if missing, don't auto-upgrade |
| `always-upgrade-to-latest-minor` | Upgrade to latest on every run |
| `absent` | Remove the packages. Configuration, user and service tasks are skipped. |

## Upgrade

```bash
ansible-playbook -i <inventory> playbook.yml -e clickhouse_upgrade=always-upgrade-to-latest-minor --tags warehouse-pg-clickhouse
```

To pin a version: `-e clickhouse_version=24.3.1`.

## Bring your own ClickHouse

This role is not needed when ClickHouse is installed elsewhere. Set `wem_clickhouse_url` directly in the inventory, see the main [README](../../README.md).
