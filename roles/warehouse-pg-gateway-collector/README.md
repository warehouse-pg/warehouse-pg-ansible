# warehouse-pg-gateway-collector

Installs `edb-otelcol` from the EDB repository on the hosts in the `monitoring_stack` inventory group (the same hosts as ClickHouse) and configures it as a central OTLP gateway. It receives telemetry from the cluster node agents and forwards it to ClickHouse.

The role does nothing unless `gateway_collector_enabled` is `true` for the host. It is off by default.

Recommended when ClickHouse is shared by multiple WarehousePG clusters or other services: the gateway gives all of them a single OTLP endpoint in front of ClickHouse.

## Run

```bash
ansible-playbook -i inventory-with-monitoring.yml playbook.yml --tags warehouse-pg-gateway-collector
```

## Where the role runs

The role runs on the `monitoring_stack` group, which holds the host selected by `monitoring_host` (falls back to `wem_host` when `monitoring_host` is not set). The `edb-repository` role runs first on the same host.

Enable it in the inventory, on the `monitoring_host` entry or in the cluster vars:

```yaml
gateway_collector_enabled: true
```

The role supports Red Hat based systems only.

## What the role configures

The role overwrites the RPM-shipped `/etc/edb/otelcol/config.yaml` completely. The file is fully managed by Ansible and is written with mode `0640` (owner `edb-otelcol`), because it embeds the ClickHouse password.

| Part | Setting |
|---|---|
| Receivers | OTLP over gRPC (`otelcol_gateway_grpc_endpoint`, default `0.0.0.0:4317`) and HTTP (`otelcol_gateway_http_endpoint`, default `0.0.0.0:4318`) |
| Processor | `batch` (`otelcol_batch_timeout`, `otelcol_batch_send_size`) |
| Exporter `clickhouse` | Metrics and logs, to `otelcol_clickhouse_endpoint` and database `otelcol_clickhouse_database`. Retries on failure for up to 300s. |
| Exporter `prometheus` | Metrics only, scrape endpoint `otelcol_prometheus_endpoint` |

The `prometheus` exporter lets a Prometheus server pull the metrics in addition to the ClickHouse export:

| Setting | Value | Meaning |
|---|---|---|
| `endpoint` | `otelcol_prometheus_endpoint`, default `127.0.0.1:8889` | Address and port of the `/metrics` endpoint. The default is loopback only, so Prometheus must run on the same host. Set it in inventory (e.g. `"<private-ip>:8889"`) to allow scraping from another host. |
| `namespace` | `whpg` | Prefix added to every metric name, e.g. `whpg_<metric>`. |
| `send_timestamps` | `true` | Each sample carries the timestamp of the original OTLP data point rather than the scrape time. |
| `metric_expiration` | `180s` | A series without new data for 3 minutes is dropped from the endpoint. |
| `resource_to_telemetry_conversion` | `enabled: true` | OTLP resource attributes (e.g. `host.name`, `service.name`) become labels on every metric. |

## Prerequisites

- *EDB repository* configured on the target host (done by `edb-repository` in the playbook).
- ClickHouse reachable from the gateway. By default the gateway writes to ClickHouse on `127.0.0.1:9000`, so ClickHouse must run on the same host (role `warehouse-pg-clickhouse`).
- `clickhouse_user` and `clickhouse_password` set in the inventory (same values as for the `warehouse-pg-clickhouse` role). Pre-flight fails if `otelcol_clickhouse_endpoint` contains no user or password.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `gateway_collector_enabled` | `false` | Install and configure the gateway collector. |
| `otelcol_gateway_package` | `edb-otelcol` | Package name. |
| `otelcol_gateway_version` | `""` | Pin a specific version (e.g. `0.158.0`). Leave empty to install whatever the EDB repository provides. |
| `otelcol_gateway_upgrade` | `install-but-no-automatic-upgrades` | Install/upgrade/remove policy, see below. |
| `otelcol_gateway_grpc_endpoint` | `0.0.0.0:4317` | OTLP gRPC receiver endpoint. |
| `otelcol_gateway_http_endpoint` | `0.0.0.0:4318` | OTLP HTTP receiver endpoint. |
| `otelcol_clickhouse_endpoint` | _(derived)_ | `clickhouse://<clickhouse_user>:<clickhouse_password>@<clickhouse_host>:<clickhouse_tcp_port>`, with `clickhouse_host` defaulting to `127.0.0.1` and `clickhouse_tcp_port` to `9000`. Override only when ClickHouse runs on a different host than the gateway. |
| `otelcol_clickhouse_database` | `acp_observability` | ClickHouse database the telemetry is written to. |
| `otelcol_clickhouse_timeout` | `5s` | ClickHouse exporter timeout. |
| `otelcol_prometheus_endpoint` | `127.0.0.1:8889` | Prometheus scrape endpoint. |
| `otelcol_batch_timeout` | `5s` | Batch processor timeout. |
| `otelcol_batch_send_size` | `8192` | Batch processor send size. |
| `otelcol_gateway_config_file` | `/etc/edb/otelcol/config.yaml` | Collector configuration file. |
| `otelcol_gateway_service_name` | `edb-otelcol` | systemd service name. |
| `otelcol_gateway_service_started` | `true` | Whether the service is started (`true`) or stopped (`false`). |
| `DNF_TIMEOUT` | `120` | Maximum time (seconds) to wait for the RPM repository. |

## Package state

| Value of `otelcol_gateway_upgrade` | Effect |
|---|---|
| `install-but-no-automatic-upgrades` (default) | Install if missing, don't auto-upgrade |
| `always-upgrade-to-latest-minor` | Upgrade to latest on every run |
| `absent` | Remove the package. Configuration and service tasks are skipped. |

## Upgrade

```bash
ansible-playbook -i <inventory> playbook.yml -e otelcol_gateway_upgrade=always-upgrade-to-latest-minor --tags warehouse-pg-gateway-collector
```

To pin a version: `-e otelcol_gateway_version=0.158.0`.

## WEM integration

When `gateway_collector_enabled` is `true`, the dashboard role derives `gateway_collector_endpoint` as `http://<monitoring_host_ip>:4317` and writes it to `OTEL_GATEWAY_ENDPOINT` in `wem.conf`. See the main [README](../../README.md).
