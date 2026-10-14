# WarehousePG Ansible Roles

Ansible roles for deploying and managing WarehousePG clusters.

## Roles

### edb-repository

Installs the EDB package repository on Red Hat based systems. This role must run before `warehouse-pg`, in order to make the WarehousePG packages available.

See [roles/edb-repository/README.md](roles/edb-repository/README.md) for details.

#### EDB Token

This role requires an EDB token and a repository name.

Set the variable `$edb_token` with the token, and the variable `$edb_repository` with the repository name.

The repository name defaults to `dev`, can also be `gsupp` or another available repository.
Ask your EDB contact for details.

One example of how to read token and repository from a file on the Ansible controller, and add the variables to the inventory:

```
- name: Read EDB token and repository from local files
  hosts: all
  gather_facts: false
  tags: always
  tasks:
    - name: Read token from local file
      ansible.builtin.set_fact:
        edb_token: "{{ lookup('file', lookup('env', 'HOME') ~ '/.edb-token') }}"
        edb_repository: "{{ lookup('file', lookup('env', 'HOME') ~ '/.edb-repository') }}"
      run_once: true
```

You don't have to follow this example, and can set the `$edb_token` and `$edb_repository` variables differently in your inventory.

#### Installation process

These roles must run as root on the remote systems, and therefore require Ansible `become` privileges (usually `sudo`).

The `edb-repository` role will download a script which sets up the repository. If you do not want to run arbitrary scripts (`setup.sh`) as root, exclude this role and setup the repository manually.

### warehouse-pg

Deploys and configures a WarehousePG cluster. Handles pre-flight validation, OS-level user/group setup, SSH key exchange, package installation, data directory calculation, and cluster hostfile generation.

**Task flow** (`roles/warehouse-pg/tasks/main.yml`):

1. `00-acl.yml`: Installs the `acl` package, required for `become_user` to an unprivileged user
2. `01-pre-flight-checks.yml`: Validates all required and optional variables
3. `02-unix-user.yml`: Creates the user and group specified in `$WHPG_UNIX_USER` and `$WHPG_UNIX_GROUP` (usually `gpadmin:gpadmin`)
4. `03-hostfiles.yml`: Generates per-cluster hostfiles in the Unix user's home directory
5. `04-os.yml`: Updates the `/etc/hosts` files on all cluster systems and prepares the OS (for example cgroups for resource groups)
6. `05-ssh-keyexchange.yml`: Exchanges SSH keys across the cluster
7. `06-install.yml`: Installs WarehousePG and optional backup packages, calculates data directories
8. `07-initialize-database.yml`: Initializes the WarehousePG database, applies `pg_hba.conf` customizations, reconciles the standby and syncs resource groups or queues (coordinator only)
9. `08-extensions.yml`: Installs or removes the optional extension packages
10. `10-roles.yml`: Creates or updates database roles from `whpg_db_roles` (tag `whpgroles`)
11. `09-expand-cluster.yml`: Expands the cluster when new segment hosts are added and `whpg_expand_db` is `true` (coordinator only)

`11-update-pg-hba.yml` is not part of this flow. It is a standalone task file which updates `pg_hba.conf` on the coordinator, used by the `warehouse-pg-wem-dashboard` role and by `--tags update-pg-hba`.

### warehouse-pg-cluster-data

Snapshots the state of every cluster declared as a child of `whpg_clusters`. Runs on each cluster's coordinator (`whpg_role: coordinator`); other hosts short-circuit out. Captures the database version, global cluster localization, connection / FTS / interconnect / network tuning / memory GUCs, OS-level memory tuning, segment topology, resource manager configuration (with the active groups or queues exported), non-default GUCs, tablespaces with per-segment locations, all databases, available and installed extensions, the coordinator's `pg_hba.conf` (comment lines stripped), and an inventory of the cluster hostfiles on disk.

Writes two files into the Unix user's home directory on each coordinator:

```
~/cluster_state_<cluster_name>.txt   # human-readable
~/cluster_state_<cluster_name>.json  # machine-readable
```

Can be run manually on its own via `ansible-playbook cluster-data.yml`, or as part of the general installation. Make sure it runs near the end, to capture the state after the installation.

> **Note:** This role is **not idempotent**. It rewrites both `cluster_state_<cluster_name>.{txt,json}` files on every run; the `scanned_at` timestamp changes each time, and any drift in the cluster state will surface in the file contents. Expect every Ansible run to report this role as `changed`.

### warehouse-pg-wem-dashboard

Installs the WarehousePG Enterprise Manager (WEM) on the host defined by `wem_host` - a monitoring and administration dashboard for WarehousePG. The role installs the `whpg-enterprise-manager` RPM, edits the RPM-shipped `/etc/edb/wem/wem.conf` in place (preserving the shipped commented examples and any RPM-generated secrets), and enables the `wem` systemd service. Default user/password for WEM is admin/admin. WEM should be accessible at http://<wem_host_ip>:8080/, provided firewall and security group settings are modified correctly.

See [roles/warehouse-pg-wem-dashboard/README.md](roles/warehouse-pg-wem-dashboard/README.md) for variables, prerequisites, and details.

### warehouse-pg-wem-state-database

Installs PostgreSQL 15 on the host defined by `wem_host` via the DNF AppStream module. After installation, the role creates the WEM state database and user (`wem_admin`) and updates `pg_hba.conf` to use trust auth for loopback connections (127.0.0.1 and ::1). This role must run before `warehouse-pg-wem-dashboard`.

See [roles/warehouse-pg-wem-state-database/README.md](roles/warehouse-pg-wem-state-database/README.md) for variables and details.

### warehouse-pg-clickhouse

Installs `edb-clickhouse-server` and `edb-clickhouse-client` from the EDB repository on the hosts in the `monitoring_stack` inventory group. Configures `listen_host` (loopback plus the host's private `ip_address`), `http_port`, `tcp_port`, and logger settings via `config.d/`. Creates the `clickhouse_user` user with `clickhouse_password` in `users.d/`, and restricts the shipped `default` user (empty password) to loopback connections only. Starts the service via `clickhouse start`.

See [roles/warehouse-pg-clickhouse/README.md](roles/warehouse-pg-clickhouse/README.md) for variables, prerequisites, and details.

### warehouse-pg-gateway-collector

Installs `edb-otelcol` from the EDB repository on the hosts in the `monitoring_stack` inventory group (the same hosts as ClickHouse) and configures it as a central OTLP gateway. Receives telemetry from all cluster node agents over gRPC (port 4317) and HTTP (port 4318) and forwards it to ClickHouse over `127.0.0.1`. Only deployed when `gateway_collector_enabled: true` is set for the `monitoring_stack` host in inventory (on its host entry or in the cluster vars).

Recommended when ClickHouse is shared by multiple WarehousePG clusters or by other services: the gateway gives all of them a single OTLP endpoint in front of ClickHouse. 

The gateway also exposes the metrics it receives on a Prometheus scrape endpoint (`prometheus` exporter), so a Prometheus server can pull them in addition to the ClickHouse export:

| Setting | Value | Meaning |
|---|---|---|
| `endpoint` | `otelcol_prometheus_endpoint`, default `127.0.0.1:8889` | Address and port the `/metrics` endpoint listens on. The default is loopback only, so Prometheus must run on the same host; set `otelcol_prometheus_endpoint` in inventory (e.g. `"<private-ip>:8889"`) to allow scraping from another host. |
| `namespace` | `whpg` | Prefix added to every metric name, e.g. `whpg_<metric>`. |
| `send_timestamps` | `true` | Each sample carries the timestamp from the original OTLP data point rather than the scrape time. |
| `metric_expiration` | `180s` | A series that receives no new data for 3 minutes is dropped from the endpoint, so stopped nodes or removed metrics disappear. |
| `resource_to_telemetry_conversion` | `enabled: true` | OTLP resource attributes (e.g. `host.name`, `service.name`) are added as labels on every metric, so series can be filtered per node. |

See [roles/warehouse-pg-gateway-collector/README.md](roles/warehouse-pg-gateway-collector/README.md) for variables, prerequisites, and details.

### acp-host-agent

Installs the `edb-acp-host-agent` RPM on all cluster nodes (coordinator, standby, and segments). Connects to WEM and registers itself.

See [roles/acp-host-agent/README.md](roles/acp-host-agent/README.md) for variables and details.

---

## What gets installed where

The table shows what the playbook installs for `inventory-with-monitoring.yml`: one cluster (`cluster_test`) with a coordinator and segments, plus a separate `monitoring_instance` host that is in both `wem_stack` and `monitoring_stack`. Other inventories change the hosts (for example a standby adds one more host to the `warehouse-pg` and `acp-host-agent` rows), but not the roles.

| Component | Role | Hosts | Condition |
|---|---|---|---|
| WarehousePG | `warehouse-pg` | all hosts in `whpg_clusters` (coordinator, standby, segments) | always |
| WEM dashboard | `warehouse-pg-wem-dashboard` | `wem_stack` (`monitoring_instance`) | `wem_stack` is not empty |
| Postgres (WEM state database) | `warehouse-pg-wem-state-database` | `wem_stack` (`monitoring_instance`) | `wem_stack` is not empty |
| ClickHouse | `warehouse-pg-clickhouse` | `monitoring_stack` (`monitoring_instance`) | `monitoring_stack` is not empty |
| OTel gateway collector (`edb-otelcol`) | `warehouse-pg-gateway-collector` | `monitoring_stack` (`monitoring_instance`) | `gateway_collector_enabled: true` on the monitoring host (off by default) |
| ACP host agent | `acp-host-agent` | all hosts in `whpg_clusters` (coordinator, standby, segments) | `wem_stack` is not empty |
| Temporal | none, bundled in the WEM RPM | `wem_stack` (`monitoring_instance`) | started by `wem.service`, listens on `127.0.0.1:7233` |
| Alertmanager | none, bundled in the WEM RPM | `wem_stack` (`monitoring_instance`) | started by `wem.service`, listens on `127.0.0.1:9093` |
| Loki | none | not installed | no role installs it (anymore) |
| Prometheus | none | not installed | no role installs it (anymore), the OTel gateway collector only exposes a scrape endpoint |

Notes:

- The `warehouse-pg-cluster-data` role also runs on `whpg_clusters` and collects the cluster state.
- The `edb-repository` role must run on every host that needs software from the EDB repository. Alternatively install the repository manually.
- Temporal and Alertmanager are not separate systemd services. `wem serve` starts them as child processes, so they start and stop with the `wem` service. Both listen on loopback only.
- The `alertmanager.yml` shipped with the WEM RPM (`/var/lib/edb/wem/alertmanager.yml`) routes by severity, but all email and Slack receivers are commented out. No notifications are sent until the file is edited. The roles do not manage it.

## Deployment Configuration (host_vars, group_vars)

Ansible automatically loads variables from `group_vars/` based on group membership. This is a good place to configure shared settings across roles without repeating them in every inventory file. Role `defaults/` serve as fallback values. `group_vars` will override defaults for actual deployments. Examples:

```
group_vars/
  whpg_clusters/
    vars.yml           # variables for cluster node specific deployment (ACP host agent)
  wem_stack/
    vars.yml           # variables for WEM stack hosts (PostgreSQL state database, WEM dashboard)
  monitoring_stack/
    vars.yml           # variables for monitoring hosts (ClickHouse, OTel gateway collector)
```

> **Note on groups:** `wem_stack` and `monitoring_stack` are groups defined in the inventory. Ansible loads `group_vars/wem_stack/` and `group_vars/monitoring_stack/` for the hosts in these groups like for any other group.

### All hosts (`group_vars/all/vars.yml`)

Variables here apply to every host regardless of group - `whpg_clusters`, `wem_stack`, and `monitoring_stack`. Use this for variables that are shared between the cluster plays (which create the DB role) and the WEM plays (which connect to it).

| Variable | Default | Description |
|---|---|---|
| `whpg_user` | `wem_user` | WarehousePG database user created for WEM, and the user WEM connects as. |
| `whpg_password` | _(set in inventory, required)_ | Password for `whpg_user`. No built-in default. |

> **Note:** `whpg_user` and `whpg_password` are one set of variables used by two roles: `warehouse-pg` (the `pg_hba.conf` entries for WEM on the coordinator) and `warehouse-pg-wem-dashboard` (creates the database role and connects as it). Both roles must see the same values, so set them here, or in the cluster vars (`cluster_<name>/vars`), and not only for `wem_stack`. If `whpg_user` is overridden only for `wem_stack`, the next `--tags deploy` run rewrites the `pg_hba.conf` entries for the default `wem_user` and WEM loses access.

### WEM stack hosts (`group_vars/wem_stack/vars.yml`)

Variables here apply only to hosts in the `wem_stack` group. The `wem_stack` group is defined in the inventory (see [Inventory Examples](#inventory-examples)). This covers warehouse-pg-wem-state-database and the WEM dashboard.

| Variable | Default | Description |
|---|---|---|
| `pg_port` | `5433` | warehouse-pg-wem-state-database port for WEM state database |
| `whpg_port` | `5432` | WarehousePG coordinator port. |
| `wem_state_db` | `wem` | WEM state database name |
| `wem_state_db_host` | `127.0.0.1` | WEM state database host |
| `wem_state_db_port` | `{{ pg_port }}` | WEM state database port |
| `wem_state_db_user` | `wem_admin` | WEM state database user |
| `wem_state_db_password` | _(empty)_ | WEM state database password. Leave empty to use trust auth. Required when `wem_state_db_host` is not localhost. |
| `wem_port` | `8080` | Port the WEM web interface listens on. |
| `wem_admin_password` | `admin` | Initial admin password. Reset on first login. |
| `clickhouse_host` | _(monitoring_host ip_address)_ | ClickHouse host WEM connects to. Auto-resolved from the `monitoring_host` IP (or the `wem_host` IP when `monitoring_host` is not set); this must match the host in `monitoring_stack`. Override only when ClickHouse is not installed by this playbook. |
| `clickhouse_tcp_port` | `9000` | ClickHouse native protocol port |
| `clickhouse_user` | _(none - required)_ | ClickHouse user WEM and the gateway collector connect as. Created by the `warehouse-pg-clickhouse` role, which also restricts the shipped `default` user (empty password) to loopback connections only. Letters, digits and underscore only; must not be `default`. Set in inventory on `monitoring_host` (or in the cluster vars) - the WEM play reads it from there. |
| `clickhouse_password` | _(none - required)_ | Password for `clickhouse_user`, at least 11 characters (`clickhouse_password_min_length`). Set in inventory next to `clickhouse_user`. |
| `clickhouse_listen_hosts` | `["127.0.0.1", "<ip_address>"]` | Addresses ClickHouse binds to: loopback plus the host's private `ip_address`. Do not use `ansible_host` - on cloud hosts it is the public IP. |
| `wem_clickhouse_url` | _(derived)_ | Full ClickHouse DSN written to `CLICKHOUSE_URL` in `wem.conf`. Auto-derived as `clickhouse://user:password@host:port`. **To bring your own ClickHouse**, set `wem_clickhouse_url` directly in inventory (e.g. `wem_clickhouse_url: "clickhouse://myuser:mypass@10.0.1.50:9000"`) - this overrides all individual `clickhouse_*` variables. |
| `gateway_collector_enabled` | `false` | Set to `true` for `monitoring_host` (on its host entry or in the cluster vars) to install and configure the OTel gateway collector next to ClickHouse. |
| `gateway_collector_endpoint` | _(empty)_ | OTLP gRPC endpoint written to `OTEL_GATEWAY_ENDPOINT` in `wem.conf`. Auto-derived as `http://<monitoring_host_ip>:4317` when `gateway_collector_enabled` is `true`. Override in inventory to use a different host or port. |

<details>
<summary>Full example: <code>group_vars/wem_stack/vars.yml</code></summary>

```yaml
---

# =============================================================================
# Shared configuration for the wem_stack group
# Applies to all hosts in wem_stack (wem-state-database, wem-dashboard).
# Role defaults serve as fallback - these values override them for deployments.
# =============================================================================

# -----------------------------------------------------------------------------
# PostgreSQL state database settings (warehouse-pg-wem-state-database)
# -----------------------------------------------------------------------------
pg_port: 5433

# -----------------------------------------------------------------------------
# WEM dashboard (warehouse-pg-wem-dashboard)
# -----------------------------------------------------------------------------
# Port the WEM web interface listens on. Leave it commented to use the default 8080.
# wem_port: 8080
# Initial password for the WEM web interface for user 'admin' (default: admin).
# Reset it on first login.
wem_admin_password: "admin"
#
# -------------------------------------------------------------
# WHPG Database connection settings for WEM dashboard.
# -------------------------------------------------------------
# If coordinator is in inventory, its IP is used automatically as whpg_host. 
# If not, whpg_host from here will be used, or -e whpg_host= is used.
whpg_host: "localhost"

# WHPG user and password for WEM dashboard to connect to the WarehousePG cluster.
# whpg_user and whpg_password are shared with the warehouse-pg role (pg_hba.conf
# entries on the coordinator). Set them in group_vars/all/vars.yml or the cluster
# vars, not here. See "All hosts" above.
# whpg_user: "wem_user"
# whpg_password: ""

# WHPG port for WEM dashboard to connect to the WarehousePG cluster.
whpg_port: 5432

# WHPG database for WEM dashboard to connect to the WarehousePG cluster.
# The database should be accessible for whpg_user mentioned above.
# If not provided/commented, the WEM dashboard will attempt to connect to the 
# the default postgres database on the WarehousePG cluster and create wem database.
# whpg_database: "wem"

# Optional existing separate monitoring database for WEM to monitor
# the WarehousePG cluster. If not provided/commented, the WEM dashboard 
# will reuse whpg_database for monitoring.
# whpg_monitoring_db: "wem_monitoring"

# -------------------------------------------------------------
# WEM state database connection settings for WEM dashboard
# -------------------------------------------------------------
# Postgresql database connection settings for WEM dashboard to connect to the WEM state database.
wem_state_db_host: "127.0.0.1"
wem_state_db_port: "{{ pg_port }}"
wem_state_db_user: "wem_admin"
# Leave empty to use trust auth (pg_hba.conf must allow it and postgres is installed locally)
# wem_state_db_password: ""
# Optional existing WEM state database for WEM dashboard to connect to.
# If not provided/commented, the WEM dashboard will attempt to connect to the
# the default postgres database on the WarehousePG cluster and create wem database.
# wem_state_db: "wem"

# -------------------------------------------------------------
# ClickHouse connection for WEM dashboard
# -------------------------------------------------------------
clickhouse_host: "127.0.0.1"
clickhouse_tcp_port: 9000
clickhouse_user: "<clickhouse-user>"           # required, not "default"
clickhouse_password: "<clickhouse-password>"   # required, at least 11 characters

wem_clickhouse_url: "clickhouse://{{ clickhouse_user }}:{{ clickhouse_password }}@{{ clickhouse_host }}:{{ clickhouse_tcp_port }}"
```

</details>

### Cluster nodes (`group_vars/whpg_clusters/vars.yml`)

Variables here apply to all hosts in the `whpg_clusters` group - coordinator, standby, and segment nodes. This covers the ACP host agent, which is deployed on every cluster node.

The `acp-host-agent` role derives `wem_connect_host` and `wem_port` from `wem_host` set in the inventory `cluster_<name>/vars` section (where `<name>` matches `cluster_name`). Override for a single run with `-e wem_host=<host>`. If not set, the defaults (`127.0.0.1`, port `8080`) apply.

| Variable | Default | Description |
|---|---|---|
| `wem_connect_host` | `127.0.0.1` | Host where WEM is running. Derived from `wem_host` set in inventory (`cluster_<name>/vars`); falls back to `127.0.0.1` when not set. Override per-run with `-e wem_host=<host>`. |
| `wem_connect_address` | `{{ wem_connect_host }}:{{ wem_port + 1 }}` | WEM relay address for the ACP host agent (WEM port + 1) |
| `wem_task_queues` | auto-derived from `whpg_role` | Task queue(s) the agent polls. Defaults: coordinator → `coordinator-tasks`, standby → `standby-tasks`. For segments no value is set and the agent uses its built-in default. Override in inventory to customise. |

<details>
<summary>Full example: <code>group_vars/whpg_clusters/vars.yml</code></summary>

```yaml
---

# =============================================================================
# Shared configuration for node-level agents (acp-host-agent)
# Applies to all hosts in whpg_clusters.
# =============================================================================

greenplum_path: "/usr/edb/whpg7/greenplum_path.sh"

# -----------------------------------------------------------------------------
# ACP host agent (acp-host-agent)
# -----------------------------------------------------------------------------
# IP of the host running WEM. Derived from wem_host in the role defaults;
# falls back to 127.0.0.1. The agent connects to WEM's relay port (wem_port + 1).
# wem_connect_host: "127.0.0.1"
wem_connect_address: "{{ wem_connect_host }}:{{ wem_port | int + 1 }}"

# Task queues - auto-derived from whpg_role (see defaults/main.yml).
# Override here or in inventory to customise per node.
# wem_task_queues: "coordinator-tasks"
```

</details>

---

## Inventory Examples

`wem_host` and `monitoring_host` are set per-cluster in `cluster_test/vars` (not at `all/vars` level). They can be overridden per-run with `-e wem_host=<host>` or `-e monitoring_host=<host>`.

All clusters are defined as children of `whpg_clusters`. Each host must have:
- `whpg_role`: one of `coordinator`, `standby`, or `segment`
- `ip_address`: private IP used to derive WEM endpoint URLs at play-level

`wem_host` and `monitoring_host` select where each part of the stack runs:

| Variable | Matching inventory group | What runs there |
|---|---|---|
| `wem_host` | `wem_stack` | WEM dashboard and its PostgreSQL state database |
| `monitoring_host` | `monitoring_stack` | ClickHouse and the OTel gateway collector. |

Set both to the same host to run everything on one host, or to different hosts to keep ClickHouse off the WEM host. WEM connects to ClickHouse and the gateway collector on the `monitoring_host` private IP.

Three separate constructs work together:

| Construct | Defined where | Purpose |
|---|---|---|
| `wem_host` / `monitoring_host` | `cluster_<name>/vars` | Name of the host whose IP address WEM, ClickHouse and the agents use. |
| `wem_stack` / `monitoring_stack` | Static inventory groups | The hosts the WEM and ClickHouse / gateway collector plays run on. |
| `whpg_clusters_monitoring` | Static inventory group | Connection details (`ansible_host`, `ip_address`, `hostname`) for a dedicated host that is not in `whpg_clusters`, so that `hostvars` is populated for it. The name of the child group below it (`monitoring_host_for_test` in the examples) is arbitrary. |

> **Important:** the `wem_stack` and `monitoring_stack` groups are **not** derived from `wem_host` and `monitoring_host`. Define them in the inventory, each containing exactly the host named by the matching variable (`wem_stack` with `wem_host`, `monitoring_stack` with `monitoring_host`). Without the groups the plays are skipped: no WEM, ClickHouse, gateway collector or ACP host agent is deployed. The ACP host agent and the `pg_hba.conf` update for WEM only run when `wem_stack` is not empty.

When the standby doubles as the WEM or monitoring host it is already in `whpg_clusters` with full connection details, so `whpg_clusters_monitoring` is not needed. The host still has to be listed in `wem_stack` and `monitoring_stack`.

### inventory: cluster only

Just a WarehousePG installation with one cluster, one coordinator host and two segment hosts.

No `wem_host`, `monitoring_host`, `wem_stack` or `monitoring_stack` defined. Without a `wem_stack` group, no WEM stack, ClickHouse or ACP host agent is deployed.

<details>
<summary>Show example</summary>

```yaml
all:
  children:
    whpg_clusters:
      children:
        cluster_test:
          vars:
            cluster_name: test
            whpg_cluster_name: test
            whpg_data_directory: /data/whpg
            whpg_host: <coordinator-private-ip>
          hosts:
            coordinator:
              ansible_host: <coordinator-public-ip>
              whpg_role: coordinator
              ip_address: <coordinator-private-ip>
              hostname: coordinator
            seg1:
              ansible_host: <seg1-public-ip>
              whpg_role: segment
              ip_address: <seg1-private-ip>
              hostname: seg1
            seg2:
              ansible_host: <seg2-public-ip>
              whpg_role: segment
              ip_address: <seg2-private-ip>
              hostname: seg2
```

</details>

### inventory: cluster with standby

Coordinator, standby, two segment hosts.

<details>
<summary>Show example</summary>

```yaml
all:
  children:
    whpg_clusters:
      children:
        cluster_test:
          vars:
            cluster_name: test
            whpg_cluster_name: test
            whpg_data_directory: /data/whpg
            whpg_host: <coordinator-private-ip>
          hosts:
            coordinator:
              ansible_host: <coordinator-public-ip>
              whpg_role: coordinator
              ip_address: <coordinator-private-ip>
              hostname: coordinator
            standby:
              ansible_host: <standby-public-ip>
              whpg_role: standby
              ip_address: <standby-private-ip>
              hostname: standby
            seg1:
              ansible_host: <seg1-public-ip>
              whpg_role: segment
              ip_address: <seg1-private-ip>
              hostname: seg1
            seg2:
              ansible_host: <seg2-public-ip>
              whpg_role: segment
              ip_address: <seg2-private-ip>
              hostname: seg2
```

</details>

### inventory: dedicated monitoring node

`wem_host` and `monitoring_host` are set to `monitoring_instance` in `cluster_test/vars`. The `wem_stack` and `monitoring_stack` groups both contain that host. A static `whpg_clusters_monitoring` group defines its connection details so `hostvars` is populated. Set `clickhouse_user` and `clickhouse_password` on the `monitoring_instance` host entry (the `monitoring_host`).

<details>
<summary>Show example</summary>

```yaml
all:
  children:
    monitoring_stack:
      hosts:
        monitoring_instance: {}
    wem_stack:
      hosts:
        monitoring_instance: {}
    whpg_clusters:
      children:
        cluster_test:
          vars:
            cluster_name: test
            whpg_cluster_name: test
            whpg_data_directory: /data/whpg
            whpg_host: <coordinator-private-ip>
            whpg_password: <whpg-password>   # required - password for whpg_user (WEM connects as it)
            wem_host: monitoring_instance
            monitoring_host: monitoring_instance
          hosts:
            coordinator:
              ansible_host: <coordinator-public-ip>
              whpg_role: coordinator
              ip_address: <coordinator-private-ip>
              hostname: coordinator
            seg1:
              ansible_host: <seg1-public-ip>
              whpg_role: segment
              ip_address: <seg1-private-ip>
              hostname: seg1
    whpg_clusters_monitoring:
      children:
        monitoring_host_for_test:
          hosts:
            monitoring_instance:
              ansible_host: <monitoring-public-ip>
              ip_address: <monitoring-private-ip>
              hostname: monitoring
              clickhouse_user: <clickhouse-user>           # required, not "default"
              clickhouse_password: <clickhouse-password>   # required, at least 11 characters
```

</details>

### inventory: standby + dedicated monitoring node

<details>
<summary>Show example</summary>

```yaml
all:
  children:
    monitoring_stack:
      hosts:
        monitoring_instance: {}
    wem_stack:
      hosts:
        monitoring_instance: {}
    whpg_clusters:
      children:
        cluster_test:
          vars:
            cluster_name: test
            whpg_cluster_name: test
            whpg_data_directory: /data/whpg
            whpg_host: <coordinator-private-ip>
            whpg_password: <whpg-password>   # required - password for whpg_user (WEM connects as it)
            wem_host: monitoring_instance
            monitoring_host: monitoring_instance
          hosts:
            coordinator:
              ansible_host: <coordinator-public-ip>
              whpg_role: coordinator
              ip_address: <coordinator-private-ip>
              hostname: coordinator
            standby:
              ansible_host: <standby-public-ip>
              whpg_role: standby
              ip_address: <standby-private-ip>
              hostname: standby
            seg1:
              ansible_host: <seg1-public-ip>
              whpg_role: segment
              ip_address: <seg1-private-ip>
              hostname: seg1
    whpg_clusters_monitoring:
      children:
        monitoring_host_for_test:
          hosts:
            monitoring_instance:
              ansible_host: <monitoring-public-ip>
              ip_address: <monitoring-private-ip>
              hostname: monitoring
              clickhouse_user: <clickhouse-user>           # required, not "default"
              clickhouse_password: <clickhouse-password>   # required, at least 11 characters
```

</details>

### inventory: standby doubles as WEM/monitoring host

The standby is already in `whpg_clusters` with full connection details. `wem_host` and `monitoring_host` point to `standby` in the cluster group vars, and the standby is listed in `wem_stack` and `monitoring_stack`. No `whpg_clusters_monitoring` group is needed.

<details>
<summary>Show example</summary>

```yaml
all:
  children:
    monitoring_stack:
      hosts:
        standby: {}
    wem_stack:
      hosts:
        standby: {}
    whpg_clusters:
      children:
        cluster_test:
          vars:
            cluster_name: test
            whpg_cluster_name: test
            whpg_data_directory: /data/whpg
            whpg_host: <coordinator-private-ip>
            whpg_password: <whpg-password>   # required - password for whpg_user (WEM connects as it)
            wem_host: standby
            monitoring_host: standby
            clickhouse_user: <clickhouse-user>           # required, not "default"
            clickhouse_password: <clickhouse-password>   # required, at least 11 characters
          hosts:
            coordinator:
              ansible_host: <coordinator-public-ip>
              whpg_role: coordinator
              ip_address: <coordinator-private-ip>
              hostname: coordinator
            standby:
              ansible_host: <standby-public-ip>
              whpg_role: standby
              ip_address: <standby-private-ip>
              hostname: standby
            seg1:
              ansible_host: <seg1-public-ip>
              whpg_role: segment
              ip_address: <seg1-private-ip>
              hostname: seg1
```

</details>

### inventory: split WEM and monitoring hosts

WEM (dashboard + state database) runs on `standby`, so `wem_stack` contains `standby`. ClickHouse and the gateway collector run on `monitoring_instance`, so `monitoring_stack` contains `monitoring_instance`. WEM connects to ClickHouse on `<monitoring-private-ip>:9000` and to the gateway on `<monitoring-private-ip>:4317`, using the `clickhouse_user` / `clickhouse_password` set on `monitoring_instance`.

<details>
<summary>Show example</summary>

```yaml
all:
  children:
    monitoring_stack:
      hosts:
        monitoring_instance: {}
    wem_stack:
      hosts:
        standby: {}
    whpg_clusters:
      children:
        cluster_test:
          vars:
            cluster_name: test
            whpg_cluster_name: test
            whpg_data_directory: /data/whpg
            whpg_host: <coordinator-private-ip>
            whpg_password: <whpg-password>         # required - password for whpg_user (WEM connects as it)
            wem_host: standby                      # WEM dashboard + state database
            monitoring_host: monitoring_instance   # ClickHouse + gateway collector
          hosts:
            coordinator:
              ansible_host: <coordinator-public-ip>
              whpg_role: coordinator
              ip_address: <coordinator-private-ip>
              hostname: coordinator
            standby:
              ansible_host: <standby-public-ip>
              whpg_role: standby
              ip_address: <standby-private-ip>
              hostname: standby
            seg1:
              ansible_host: <seg1-public-ip>
              whpg_role: segment
              ip_address: <seg1-private-ip>
              hostname: seg1
    whpg_clusters_monitoring:
      children:
        monitoring_host_for_test:
          hosts:
            monitoring_instance:
              ansible_host: <monitoring-public-ip>
              ip_address: <monitoring-private-ip>
              hostname: monitoring
              clickhouse_user: <clickhouse-user>           # required, not "default"
              clickhouse_password: <clickhouse-password>   # required, at least 11 characters
              gateway_collector_enabled: true              # optional
```

</details>

---

## Usage

Create an Ansible inventory which defines the roles for each server. See the `Inventory Example` section earlier.

### Playbook tags

The playbook uses tags to allow selective deployment of different parts of the stack:

| Tag | Hosts | Requires | What runs |
|---|---|---|---|
| `always` | `all` | Any inventory | Reads the EDB token and repository from the local files (see [EDB Token](#edb-token)). Runs with every tag selection, and cannot be skipped by selecting other tags. |
| `deploy` | `whpg_clusters` | Any inventory | Full cluster deployment: the `/data/whpg` directory, the `edb-repository` and `warehouse-pg` roles (OS prep, packages, cluster init, `pg_hba.conf`, standby, resource management, extensions, database roles from `whpg_db_roles`, expansion) |
| `whpgroles` | `whpg_clusters` (acts on the coordinator) | Any inventory | Only the `warehouse-pg` task that creates or updates the database roles from `whpg_db_roles`. Also part of `deploy`. See [Database users](#database-users). |
| `cluster-data` | `whpg_clusters` | Any inventory | Snapshot cluster state to `cluster_state_<name>.{txt,json}` |
| `update-pg-hba` | `wem_stack` (delegates to the coordinator) | Inventory with a `wem_stack` group | Creates the WEM database role (`whpg_user`, superuser) and adds the matching `pg_hba.conf` entries on the coordinator, without a full deploy. Discovers paths via `psql`. Runs the same tasks as `warehouse-pg-db-roles`. |
| `warehouse-pg-db-roles` | `wem_stack` (delegates to the coordinator) | Inventory with a `wem_stack` group | Same tasks as `update-pg-hba`: the WEM database role and its `pg_hba.conf` entries. Only this one role is managed, other roles come from `whpg_db_roles` (tag `whpgroles`). |
| `warehouse-pg-clickhouse` | `monitoring_stack` | Inventory with a `monitoring_stack` group | Install and configure ClickHouse on the monitoring host |
| `warehouse-pg-gateway-collector` | `monitoring_stack` | Inventory with a `monitoring_stack` group + `gateway_collector_enabled: true` | Install and configure OTel gateway collector on the monitoring host |
| `wem` | `wem_stack` | Inventory with a `wem_stack` group | Deploy all WEM components: PostgreSQL state database, WEM dashboard, including the WEM database role and `pg_hba.conf` entries. Does not include ClickHouse, the gateway collector or the ACP host agent. |
| `warehouse-pg-wem-state-database` | `wem_stack` | Inventory with a `wem_stack` group | Install PostgreSQL state database only |
| `warehouse-pg-wem-dashboard` | `wem_stack` | Inventory with a `wem_stack` group | Install WEM dashboard, including the WEM database role and `pg_hba.conf` entries |
| `acp-host-agent` | `whpg_clusters` (all nodes) | Inventory with a `wem_stack` group (and `wem_host`) | Install ACP host agent on all cluster nodes |

Notes on tags:

- Tags can be combined, for example `--tags "wem,warehouse-pg-clickhouse"`.
- `update-pg-hba` and `warehouse-pg-db-roles` select the same tasks. They are also run by `wem` and `warehouse-pg-wem-dashboard`.
- `whpgroles` and `warehouse-pg-db-roles` are different: `whpgroles` applies `whpg_db_roles` from the inventory, `warehouse-pg-db-roles` only handles the role WEM connects as.
- Every `--tags` run also runs the `always` play and the `Validate wem_host` / `Validate monitoring_host` checks for the selected tags.

### Examples

#### Full stack deployment (no `--tags`)

Run without tags - what deploys depends entirely on the inventory file. ClickHouse, WEM, and ACP host agent are deployed automatically when the inventory has a `wem_stack` group (and a `monitoring_stack` group for ClickHouse).

> **Note:** This also runs the `cluster-data` play on every execution, which is not idempotent (rewrites snapshot files each time). For repeated runs, prefer the explicit-tags approach in [Selective deployment](#selective-deployment---tags).

| Inventory | Command | What deploys |
|---|---|---|
| `inventory.yml` | `ansible-playbook -i inventory.yml playbook.yml` | WarehousePG cluster (coordinator + segments) |
| `inventory-with-standby.yml` | `ansible-playbook -i inventory-with-standby.yml playbook.yml` | WarehousePG cluster (coordinator + standby + segments) |
| `inventory-with-monitoring.yml` | `ansible-playbook -i inventory-with-monitoring.yml playbook.yml` | WarehousePG cluster + ClickHouse + WEM stack (on monitoring_instance) + ACP host agent on all cluster nodes |
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml` | WarehousePG cluster (with standby) + ClickHouse + WEM stack (on monitoring_instance) + ACP host agent on all cluster nodes |
| `inventory-with-standby-as-monitoring.yml` | `ansible-playbook -i inventory-with-standby-as-monitoring.yml playbook.yml` | WarehousePG cluster (with standby) + ClickHouse + WEM (on standby) + ACP host agent on all cluster nodes |

---

#### Selective deployment (`--tags`)

##### Section 1: Cluster deployment

| Inventory | Command | What deploys |
|---|---|---|
| `inventory.yml` | `ansible-playbook -i inventory.yml playbook.yml --tags deploy` | WarehousePG cluster |
| `inventory.yml` | `ansible-playbook -i inventory.yml playbook.yml --tags cluster-data` | Snapshot cluster state only |
| `inventory-with-standby.yml` | `ansible-playbook -i inventory-with-standby.yml playbook.yml --tags deploy` | WarehousePG cluster |

All tags above work with any inventory that has `whpg_clusters` hosts defined - replace `-i` accordingly.

---

##### Section 2: WEM + ClickHouse

Use an inventory that has `wem_host` set in `cluster_test/vars` and a matching `wem_stack` group (and a `monitoring_stack` group for ClickHouse and the gateway collector).

| Inventory | Command | What deploys |
|---|---|---|
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags "wem,warehouse-pg-clickhouse"` | WEM stack + ClickHouse |
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags wem` | PostgreSQL + WEM dashboard |
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags warehouse-pg-clickhouse` | ClickHouse only |
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags warehouse-pg-wem-state-database` | PostgreSQL state DB only |
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags warehouse-pg-wem-dashboard` | WEM dashboard only |
| `inventory-with-monitoring.yml` | `ansible-playbook -i inventory-with-monitoring.yml playbook.yml --tags warehouse-pg-gateway-collector` | OTel gateway collector only (requires `gateway_collector_enabled: true` for `monitoring_host`) |

The tags in the table above work with any inventory that has `wem_host` and the `wem_stack` / `monitoring_stack` groups set - replace `-i` accordingly.

---

##### Section 3: ACP host agent

Runs on all `whpg_clusters` hosts. The ACP agent is only deployed when `wem_stack` is non-empty.

| Inventory | Command | What deploys |
|---|---|---|
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags acp-host-agent` | ACP host agent on all cluster nodes |

---

#### Upgrade

**RPM-based components** (WEM, ClickHouse, ACP host agent, WarehousePG): set the upgrade variable to `always-upgrade-to-latest-minor` and re-run the relevant tag.

| Component | Command |
|---|---|
| WarehousePG cluster | `ansible-playbook -i <inventory> playbook.yml -e whpg_upgrade=always-upgrade-to-latest-minor --tags deploy` |
| WEM dashboard | `ansible-playbook -i <inventory> playbook.yml -e wem_upgrade=always-upgrade-to-latest-minor --tags warehouse-pg-wem-dashboard` |
| ClickHouse | `ansible-playbook -i <inventory> playbook.yml -e clickhouse_upgrade=always-upgrade-to-latest-minor --tags warehouse-pg-clickhouse` |
| ACP host agent | `ansible-playbook -i <inventory> playbook.yml -e acp_agent_upgrade=always-upgrade-to-latest-minor --tags acp-host-agent` |
| Gateway collector | `ansible-playbook -i <inventory> playbook.yml -e otelcol_gateway_upgrade=always-upgrade-to-latest-minor --tags warehouse-pg-gateway-collector` |

To pin a specific RPM version, set the following variables in inventory or on the commandline:

| Component | Variable | Example |
|---|---|---|
| WEM dashboard | `wem_version` | `-e wem_version=1.2.3` |
| ClickHouse | `clickhouse_version` | `-e clickhouse_version=24.3.1` |
| ACP host agent | `acp_agent_version` | `-e acp_agent_version=1.2.3` |
| Gateway collector | `otelcol_gateway_version` | `-e otelcol_gateway_version=0.158.0` |

---

## Generated files

The role will generate a number of files in the home directory of `WHPG_UNIX_USER`:

```
~/cluster_hostfile_all_<cluster_name>            # contains all hostnames
~/cluster_hostfile_coordinator_<cluster_name>    # contains the hostname of the coordinator
~/cluster_hostfile_standby_<cluster_name>        # only if standby host is defined
~/cluster_hostfile_segments_<cluster_name>       # only if segment hosts are defined
~/gpinitsystem_config_<cluster_name>             # the gpinitsystem file used to initialize the cluster
```

---

## Database users

The `warehouse-pg` role can create or update WarehousePG database roles on the coordinator via the `whpg_db_roles` variable. Each entry creates the role if it does not exist, or updates the password and privileges if it does. Runs on the coordinator only.

```yaml
whpg_db_roles:
  - name: reporting
    password: "r3port!"
  - name: etl_admin
    password: "etl123"
    createdb: true
  - name: wem_user
    password: "admin"
    superuser: true
```

| Key | Required | Default | Description |
|---|---|---|---|
| `name` | yes | - | Role name. |
| `password` | no | - | Login password. Omit for passwordless roles. |
| `superuser` | no | `false` | Grant `SUPERUSER` privilege. |
| `createdb` | no | `false` | Grant `CREATEDB` privilege. |
| `createrole` | no | `false` | Grant `CREATEROLE` privilege. |

Run standalone (skips cluster deploy, SSH key exchange, etc.):

```bash
ansible-playbook playbook.yml --tags whpgroles
```

---

## Customizing `pg_hba.conf`

After cluster initialization, the role can extend the coordinator's `pg_hba.conf` with additional entries. Two mechanisms are available, and both can be used together. If either changes `pg_hba.conf`, the database is reloaded automatically.

### Template file per cluster

Drop a file named `pg_hba_<cluster_name>.conf` into the `warehouse-pg` role's `templates/` directory. When present, its contents are appended to the coordinator's `pg_hba.conf` inside an Ansible-managed block. The file is processed through Jinja2, so inventory variables can be referenced.

### `additional_pg_hba_conf` variable

Define `additional_pg_hba_conf` in the inventory as a list of entries. Each entry produces one `pg_hba.conf` line and the resulting block is added to the coordinator's `pg_hba.conf`.

Each entry is a hash with the following keys:

| Key | Required | Description |
|---|---|---|
| `conntype` | Yes | One of `local`, `host`, `hostssl`, `hostnossl`, `hostgssenc`, `hostnogssenc`. |
| `database` | Yes | Database name (e.g. `all`, `postgres`, `mydb`). |
| `user` | Yes | User or role name (e.g. `all`, `gpadmin`, `+admins`). |
| `address` | No | IPv4 or IPv6 address with hostmask (e.g. `10.0.0.0/8`, `2001:db8::/32`). Omit for `conntype: local`. |
| `method` | Yes | Authentication method (e.g. `trust`, `md5`, `scram-sha-256`, `peer`). |
| `options` | No | Free-form trailing options string. |

Example:

```yaml
additional_pg_hba_conf:
  - conntype: local
    database: all
    user: gpadmin
    method: peer
  - conntype: host
    database: all
    user: all
    address: 10.0.0.0/8
    method: scram-sha-256
  - conntype: hostssl
    database: mydb
    user: app_user
    address: 2001:db8::/32
    method: cert
    options: clientcert=verify-full
```

---

## Standby coordinator

The standby coordinator's lifecycle is driven by the inventory. Any host inside a cluster group that has `whpg_role: standby` is treated as the desired standby (at most one per cluster). On each run, the role reconciles inventory against the live cluster.

### Reconciliation

1. `03-hostfiles.yml` writes `~/cluster_hostfile_standby_<cluster_name>` on the coordinator when a standby host is in inventory, and deletes the file when no standby host is in inventory.
2. `07-initialize-database.yml`, on the coordinator, reads that hostfile, runs `gpstate -f`, and parses the `Standby address = …` line to find the currently configured standby.
3. The desired state (from the hostfile) and configured state (from `gpstate`) are compared:

| Inventory standby | `gpstate -f` standby | Action |
|---|---|---|
| none | none | no-op |
| `hostX` | none | `gpinitstandby -s hostX -S <whpg_standby_data_directory>/<seg_prefix>-1 -a` |
| none | `hostX` | `gpinitstandby -r -a` (remove) |
| `hostX` | `hostX` | no-op |
| `hostX` | `hostY` | fail - changing the standby hostname in place is not allowed |

### Data directory layout

The standby's data directory lives under `whpg_standby_data_directory` (default `<whpg_data_directory>/v<major_version>/standby`), with the standby data placed at `<whpg_standby_data_directory>/<seg_prefix>-1` (e.g. `/data/whpg/v7/standby/gpseg-1`). `gpinitstandby` is invoked with `-S` pointing at that path, so the standby host does **not** need the coordinator's directory path (e.g. `/data/whpg/v7/coordinator`) to exist. The parent (`whpg_standby_data_directory`) is created on the standby host by `06-install.yml`.

### Swapping the standby

To move the standby to a different host, change the inventory in two steps:

1. Remove `whpg_role: standby` from the current standby host and run the playbook. The old standby is removed via `gpinitstandby -r`.
2. Set `whpg_role: standby` on the new host and run the playbook again. The new standby is created via `gpinitstandby -s`.

Attempting both in a single inventory change (changing which host carries `whpg_role: standby` while the old standby is still configured in the database) is rejected with an explicit failure, since `gpinitstandby` does not support an in-place hostname change.

## Workload management

WarehousePG supports two mutually exclusive workload managers, selected via the variable `whpg_resource_management`. Valid values:

- `queue` - Resource Queues. Enforces concurrency, memory, and priority inside the database. No OS-level prep required.
- `group` - Resource Groups, with **auto-detection** of cgroup v1 vs v2 based on what is mounted on the host.
- `group-v2` - Resource Groups, **force cgroup v2**. Pre-flight rejects this on hosts that do not have `/sys/fs/cgroup/cgroup.controllers` (i.e. that aren't running cgroup v2 unified).

The value maps to the `gp_resource_manager` GUC. If the database currently runs under a different manager than the inventory specifies, the playbook switches it via `gpconfig` and restarts the cluster with `gpstop -r` before applying any group/queue definitions.

For Resource Groups, Greenplum 7 distinguishes between cgroup v1 (`gp_resource_manager=group`, paths under `/sys/fs/cgroup/cpu/gpdb/...`) and cgroup v2 (`gp_resource_manager=group-v2`, paths under `/sys/fs/cgroup/gpdb/` and `/sys/fs/cgroup/gpdb.service/`). With `whpg_resource_management: group`, the playbook detects which hierarchy is mounted and chooses the right GUC value for you; with `whpg_resource_management: group-v2`, it always uses `group-v2`. The OS prep in `04-os.yml` installs `libcgroup-tools` and writes `/etc/cgconfig.d/gpdb.conf` on cgroup v1 hosts, or installs a `gpdb-cgroup.service` systemd unit that creates both `/sys/fs/cgroup/gpdb` and `/sys/fs/cgroup/gpdb.service` on cgroup v2 hosts.

The pre-flight check rejects any value other than `queue`, `group`, or `group-v2`. When `group-v2` is set, pre-flight also verifies that every host has `/sys/fs/cgroup/cgroup.controllers`.

### `whpg_resource_groups`

A list of resource group definitions, applied only when `whpg_resource_management` is `group` or `group-v2`. Each entry is a hash with a unique `name` and the following attributes:

| Attribute | Type | Range / values | Notes |
|---|---|---|---|
| `CONCURRENCY` | int | 0--1000 | maximum concurrent transactions |
| `CPU_MAX_PERCENT` | int | -1, or 1--100 | mutually exclusive with `CPUSET` |
| `CPU_WEIGHT` | int | 1--500 | relative CPU priority |
| `CPUSET` | int | -1, or 0--256 | mutually exclusive with `CPU_MAX_PERCENT` |
| `IO_LIMIT` | int / str | -1, 2--4294967295, or `max` | I/O bytes per second, or `max` for unbounded |
| `MEMORY_QUOTA` | int | -1, or 0..INT_MAX (MB) | -1 = queries use `statement_mem` |
| `MIN_COST` | int | >= 0 | minimum query cost handled by the group |

If `CPUSET >= 0`, the playbook emits `CPUSET='<n>'` in the CREATE/ALTER statement (and Greenplum forces `CPU_MAX_PERCENT` to -1). If `CPUSET == -1`, the playbook emits `CPU_MAX_PERCENT=<n>` instead.

The built-in groups `default_group`, `admin_group`, and `system_group` are never removed, even if absent from inventory.

Example:

```yaml
whpg_resource_groups:
  - name: rgroup_etl              # balanced ETL workload
    CONCURRENCY: 20
    CPU_MAX_PERCENT: 20
    CPU_WEIGHT: 500
    CPUSET: -1
    IO_LIMIT: -1
    MEMORY_QUOTA: 250
    MIN_COST: 50
  - name: rgroup_reporting        # many concurrent, low-cost report queries
    CONCURRENCY: 50
    CPU_MAX_PERCENT: 30
    CPU_WEIGHT: 200
    CPUSET: -1
    IO_LIMIT: -1
    MEMORY_QUOTA: 100
    MIN_COST: 0
  - name: rgroup_admin            # dedicated CPU core for admin/maintenance
    CONCURRENCY: 5
    CPU_MAX_PERCENT: -1
    CPU_WEIGHT: 500
    CPUSET: 0
    IO_LIMIT: max
    MEMORY_QUOTA: 500
    MIN_COST: 100
```

### `whpg_resource_queues`

A list of resource queue definitions, applied only when `whpg_resource_management: queue`. Each entry is a hash with a unique `name` and:

| Attribute | Type | Values |
|---|---|---|
| `MEMORY_LIMIT` | string | PostgreSQL memory string (`'2GB'`, `'512MB'`, ...) |
| `ACTIVE_STATEMENTS` | int | > 0 |
| `PRIORITY` | string | `LOW`, `MEDIUM`, `HIGH`, `MAX` |
| `MAX_COST` | number | >= 0 |

The built-in `pg_default` queue is never removed, even if absent from inventory.

Example:

```yaml
whpg_resource_queues:
  - name: etl                     # heavy ETL: low concurrency, large memory
    MEMORY_LIMIT: '2GB'
    ACTIVE_STATEMENTS: 3
    PRIORITY: MEDIUM
    MAX_COST: 0
  - name: reporting               # many concurrent, prioritized reports
    MEMORY_LIMIT: '4GB'
    ACTIVE_STATEMENTS: 10
    PRIORITY: HIGH
    MAX_COST: 0
```

### Sync behavior

At the end of `07-initialize-database.yml`, on the coordinator host:

1. Query the current `gp_resource_manager`. If it differs from `whpg_resource_management`, run `gpconfig` and `gpstop -r` to switch.
2. Fetch the current resource groups (or queues) and their settings from the database.
3. `CREATE` any entry that appears in inventory but not in the database.
4. `ALTER` an existing entry only when at least one attribute differs from inventory - no statements are sent if everything already matches (keeps the audit log clean).
5. `DROP` any database entry not present in inventory, skipping the built-in defaults listed above.

Leaving the relevant list (`whpg_resource_groups` or `whpg_resource_queues`) at its empty default is supported. No entries will be created or updated, but the cleanup step will still drop any custom (non-default) entries that exist in the database.

See `roles/warehouse-pg/defaults/main.yml` for fully commented examples.

## Extensions

The `warehouse-pg` role can install optional WarehousePG extensions. Each extension has an associated `whpg_extension_<name>` variable that defaults to `false`. The variable accepts three kinds of values:

- `true` - install the latest package matching `$whpg_major_version`. The package state honours `whpg_upgrade` (i.e. it is upgraded to the latest minor version when `whpg_upgrade` is `always-upgrade-to-latest-minor`, otherwise only installed if missing).
- `false` - uninstall the extension package if it is currently installed.
- a string - install the exact RPM identified by the string. The string is passed verbatim to `dnf` with `state: present`, so it may be a bare package name, a `name-version-release` (NVR), a full `name-version-release.arch` (NEVRA), or a path to a local `.rpm` file. The requested version is installed, upgraded, or downgraded as needed. Note that this Playbook does not handle upgrades or downgrades in the database itself, just the package installation.

The role automatically selects the package matching `$whpg_major_version` for the `true`/`false` cases; extensions that are not available for the configured major version are skipped.

| Extension | Variable |
|---|---|
| diskquota | `whpg_extension_diskquota` |
| fts-enhanced | `whpg_extension_fts_enhanced` |
| hll | `whpg_extension_hll` |
| madlib | `whpg_extension_madlib` |
| madlib1 | `whpg_extension_madlib1` |
| pgfs | `whpg_extension_pgfs` |
| pgvector | `whpg_extension_pgvector` |
| plr | `whpg_extension_plr` |
| postgis | `whpg_extension_postgis` |
| pxf | `whpg_extension_pxf` |
| q3c | `whpg_extension_q3c` |
| user_profile | `whpg_extension_user_profile` |
| vectorchord | `whpg_extension_vectorchord` |
| whpg_fdw | `whpg_extension_whpg_fdw` |

Examples:

```yaml
# Install the latest pgvector package for the configured major version
whpg_extension_pgvector: true

# Uninstall the diskquota package if currently installed
whpg_extension_diskquota: false

# Install a specific pgvector RPM by NVR
whpg_extension_pgvector: "edb-whpg7-pgvector-0.7.4-1.el8"

# Install a specific postgis RPM by full NEVRA
whpg_extension_postgis: "edb-whpg6-postgis-3.3.2-1.el8.x86_64"

# Install madlib from a local RPM file
whpg_extension_madlib: "/var/tmp/edb-whpg7-madlib-2.1.0-1.el8.x86_64.rpm"
```

When `whpg_install_packages` is `false`, the role does not install or remove any packages and instead verifies that every extension marked with either `true` or a string value is already installed on the host (the base package name is checked, not a specific version).

This step only installs the extension package. The extension must still be activated in the database, using the `CREATE EXTENSION` command.

## All Variables

This section lists every non-internal variable referenced by the roles. Variables whose names begin with an underscore (`_`) are role-internal and are intentionally omitted.

### Shared variables (used across all roles)

| Variable | Default | Description |
|---|---|---|

### Role: `edb-repository`

| Variable | Default | Description |
|---|---|---|
| `edb_token` | `""` | EDB repository authentication token. Must be defined and non-empty. |
| `edb_repository` | `"dev"` | EDB repository name (e.g. `dev`, `gsupp`). |
| `serial_operations` | `1` | Throttle for concurrent download/install operations. |

### Role: `warehouse-pg`

#### Inventory / cluster variables (no defaults, must be provided)

| Variable | Scope | Description |
|---|---|---|
| `cluster_name` | group | Cluster identifier used to build the inventory group name (`cluster_<cluster_name>`), hostfile names, and `gpinitsystem_config_<cluster_name>`. |
| `whpg_cluster_name` | group | Cluster name identifier. Non-empty string; validated in pre-flight checks. |
| `whpg_data_directory` | group/host | Base data directory path. Must exist as a directory on every host. The role default is an empty string, which is rejected. |
| `whpg_role` | host | Role of each host within its cluster. One of `coordinator`, `standby`, `segment`. |
| `hostname` | host | Expected system hostname for the host. Compared against `ansible_facts['hostname']`; set via `hostname` module. |
| `ip_address` | host | IPv4/IPv6 address used to populate `/etc/hosts` entries for all cluster hosts. |

#### Variables with defaults

| Variable | Default | Description |
|---|---|---|
| `WHPG_UNIX_USER` | `"gpadmin"` | Unix user that owns the WarehousePG installation. |
| `WHPG_UNIX_GROUP` | `"gpadmin"` | Unix group for the WarehousePG user. |
| `whpg_major_version` | `7` | WarehousePG major version. Must be `6` or `7`. |
| `whpg_seg_prefix` | `"default"` | Segment prefix. Non-empty string. When equal to `"default"`, the template emits `gpseg`. |
| `whpg_seg_num` | `2` | Number of primary segments per host. Integer, 1--64. |
| `whpg_port` | `5432` | Coordinator listen port. Integer, 1024--65535. |
| `whpg_seg_base_port` | `6000` | Base port for primary segments (written to `PORT_BASE` in `gpinitsystem` config). Integer, 1024--65535. |
| `whpg_upgrade` | `"install-but-no-automatic-upgrades"` | Upgrade policy for warehouse-pg packages. The default `"install-but-no-automatic-upgrades"` is the safe setting: it installs the packages if missing but will not upgrade an already-installed version (maps to the `dnf` state `present`). `"always-upgrade-to-latest-minor"` auto-upgrades to the latest available minor version on every run (maps to the `dnf` state `latest`). |
| `whpg_use_versioned_directories` | `true` | When `true`, data directories include a `v<major_version>/` subdirectory. |
| `whpg_seg_enable_mirrors` | `false` | Enable mirror segments in the `gpinitsystem` configuration. |
| `whpg_seg_mirror_base_port` | `7000` | Base port for mirror segments. |
| `whpg_default_database` | `""` | Default database name. Must be a valid PostgreSQL identifier (1--63 chars, starts with letter or underscore) when non-empty. |
| `whpg_install_packages` | `true` | Whether the role should install packages. When `false`, the user is expected to install them. |
| `whpg_install_backup` | `true` | Install the `whpg-backup` package. Set to `false` to skip it. |
| `whpg_resource_management` | `"group"` | Workload manager: `queue`, `group`, or `group-v2`. Maps to the `gp_resource_manager` GUC. See the "Workload management" section. |
| `whpg_resource_groups` | `[]` | List of resource group definitions applied when `whpg_resource_management` is `group` or `group-v2`. Empty by default. |
| `whpg_resource_queues` | `[]` | List of resource queue definitions applied when `whpg_resource_management` is `queue`. Empty by default. |
| `DNF_TIMEOUT` | `300` | Maximum time (seconds) to wait for the RPM repository. |
| `serial_operations` | `1` | Throttle for concurrent package installs, to avoid hitting EDB repository retry limits. |
| `whpg_user` | `"wem_user"` | WarehousePG database user WEM connects as. Used by the `pg_hba_monitoring_wem.conf` template (`11-update-pg-hba.yml`). |
| `whpg_remove_old_gpAdminLogs` | `false` | Add maintenance jobs to remove old `gpAdminLogs` logfiles on coordinator/standby. |
| `whpg_remove_old_logs` | `false` | Add maintenance job to remove old data-directory logfiles. |
| `greenplum_path` | `"/usr/edb/whpg7/greenplum_path.sh"` | Path to the `greenplum_path.sh` environment script. |
| `whpg_extension_<name>` | `false` | Install (`true`), remove (`false`) or install a specific RPM (string) for each extension. See the "Extensions" section for the list of names. |
| `whpg_expand_db` | `true` | Automatically expand the database when segment hosts are added. Can take a long time. |
| `whpg_expand_db_redistribute` | `true` | Automatically redistribute the data after expansion. Can take a long time. |
| `whpg_expand_db_run_analyze` | `true` | Automatically run `ANALYZE` after expansion. Can take a long time. |
| `whpg_expand_cleanup_after_complete` | `true` | Automatically clean up after expansion. Only use this if you are sure the expansion works. |

#### Optional variables (no default; validated only when defined)

| Variable | Description |
|---|---|
| `additional_pg_hba_conf` | List of additional `pg_hba.conf` entries to apply on the coordinator. See the "Customizing `pg_hba.conf`" section above for the entry schema. |
| `whpg_db_roles` | List of database roles to create or update on the coordinator. See the "Database users" section for the entry schema. Empty by default. |

### Role: `warehouse-pg-cluster-data`

| Variable | Default | Description |
|---|---|---|
| `WHPG_UNIX_USER` | `"gpadmin"` | Unix user whose home directory receives the `cluster_state_<cluster_name>.{txt,json}` files. Must match the value used by `warehouse-pg`. |
| `WHPG_UNIX_GROUP` | `"gpadmin"` | Unix group of that user. |
| `greenplum_path` | `"/usr/edb/whpg7/greenplum_path.sh"` | Path to `greenplum_path.sh`, sourced before running `psql`, `gpstate` and similar commands. |

### Role: `warehouse-pg-wem-dashboard`

See [roles/warehouse-pg-wem-dashboard/README.md](roles/warehouse-pg-wem-dashboard/README.md) for the full variable reference.

### Role: `warehouse-pg-wem-state-database`

See [roles/warehouse-pg-wem-state-database/README.md](roles/warehouse-pg-wem-state-database/README.md) for the full variable reference.

### Role: `warehouse-pg-clickhouse`

| Variable | Default | Description |
|---|---|---|
| `clickhouse_server_package` | `"edb-clickhouse-server"` | Server package name. |
| `clickhouse_client_package` | `"edb-clickhouse-client"` | Client package name. |
| `clickhouse_version` | `""` | Pin a specific version (e.g. `24.3.1`). Empty installs whatever the EDB repository provides. |
| `clickhouse_upgrade` | `"install-but-no-automatic-upgrades"` | Install/upgrade/remove policy: `install-but-no-automatic-upgrades`, `always-upgrade-to-latest-minor` or `absent`. |
| `clickhouse_listen_hosts` | `["127.0.0.1", "<ip_address>"]` | Addresses ClickHouse binds to. |
| `clickhouse_http_port` | `8123` | HTTP port. |
| `clickhouse_tcp_port` | `9000` | Native protocol port. |
| `clickhouse_log_level` | `"information"` | Server log level. |
| `clickhouse_log_dir` | `"/var/log/clickhouse-server"` | Log directory. |
| `clickhouse_log_file` | `"<clickhouse_log_dir>/clickhouse-server.log"` | Main log file. |
| `clickhouse_error_log_file` | `"<clickhouse_log_dir>/clickhouse-server.err.log"` | Error log file. |
| `clickhouse_user` | _(none - required)_ | ClickHouse user to create. Must not be `default`. Set in inventory. |
| `clickhouse_password` | _(none - required)_ | Password for `clickhouse_user`. Set in inventory. |
| `clickhouse_password_min_length` | `11` | Minimum length enforced for `clickhouse_password`. |
| `clickhouse_service_name` | `"clickhouse-server"` | systemd service name. |
| `clickhouse_service_started` | `true` | Whether the role starts the service. |
| `DNF_TIMEOUT` | `120` | Maximum time (seconds) to wait for the RPM repository. |

### Role: `warehouse-pg-gateway-collector`

| Variable | Default | Description |
|---|---|---|
| `gateway_collector_enabled` | `false` | Install and configure the gateway collector. Set on the `monitoring_host` entry or in the cluster vars. |
| `otelcol_gateway_package` | `"edb-otelcol"` | Package name. |
| `otelcol_gateway_version` | `""` | Pin a specific version (e.g. `0.158.0`). Empty installs whatever the EDB repository provides. |
| `otelcol_gateway_upgrade` | `"install-but-no-automatic-upgrades"` | Install/upgrade/remove policy: `install-but-no-automatic-upgrades`, `always-upgrade-to-latest-minor` or `absent`. |
| `otelcol_gateway_grpc_endpoint` | `"0.0.0.0:4317"` | OTLP gRPC receiver endpoint. |
| `otelcol_gateway_http_endpoint` | `"0.0.0.0:4318"` | OTLP HTTP receiver endpoint. |
| `otelcol_clickhouse_endpoint` | _(derived)_ | ClickHouse DSN, built from `clickhouse_user`, `clickhouse_password`, `clickhouse_host` (default `127.0.0.1`) and `clickhouse_tcp_port`. Override only when ClickHouse runs on a different host than the gateway. |
| `otelcol_clickhouse_database` | `"acp_observability"` | ClickHouse database the telemetry is written to. |
| `otelcol_clickhouse_timeout` | `"5s"` | ClickHouse exporter timeout. |
| `otelcol_prometheus_endpoint` | `"127.0.0.1:8889"` | Address of the Prometheus scrape endpoint (`/metrics`). |
| `otelcol_batch_timeout` | `"5s"` | Batch processor timeout. |
| `otelcol_batch_send_size` | `8192` | Batch processor send size. |
| `otelcol_gateway_config_file` | `"/etc/edb/otelcol/config.yaml"` | Collector configuration file. |
| `otelcol_gateway_service_name` | `"edb-otelcol"` | systemd service name. |
| `otelcol_gateway_service_started` | `true` | Whether the role starts the service. |
| `DNF_TIMEOUT` | `120` | Maximum time (seconds) to wait for the RPM repository. |

### Role: `acp-host-agent`

See [roles/acp-host-agent/README.md](roles/acp-host-agent/README.md) for the full variable reference.

### Running upgrades

The `whpg_upgrade` variable defaults to `install-but-no-automatic-upgrades`. This is the safe setting: the playbook can be re-run at any time without upgrading installed packages, so it does not cause package changes or the database restarts that may come with them.

To upgrade, override the variable for a single run on the command line. Example:

```
ansible-playbook playbook.yml -e "whpg_upgrade=always-upgrade-to-latest-minor"
```

The new setting applies only to this run of the playbook, so updates can be applied when convenient.

## Performance

### AWS Instance Types

- Instance Type: `m7a.medium`: `1x vCPU`, `4 GB RAM`
- Instance Type: `m7a.large`: `2x vCPU`, `8 GB RAM`
- Instance Type: `m7a.xlarge`: `4x vCPU`, `16 GB RAM`
- Instance Type: `m7a.2xlarge`: `8x vCPU`, `32 GB RAM`

### Tests

Tests run with:

```
time ansible-playbook --extra-vars "whpg_seg_num=2" --extra-vars "whpg_seg_enable_mirrors=false" --extra-vars "whpg_resource_management=group-v2" playbook.yml
```

The `whpg_seg_num` parameter is adapted, see the output below.

The `forks` parameter is set to `10` (default: `10`), ssh options used:

```
ssh_args = -o ControlMaster=auto -o ControlPersist=10s -o StrictHostKeyChecking=no
```

All tests ran only once. All tests ran on the code version 2026-05-21, deploying the EDB repository (role `edb-repository`), deploying the cluster (role `warehouse-pg`), creating cluster stats (role `warehouse-pg-cluster-data`), no expansions configured, no WEM (role `warehouse-pg-wem-dashboard`).

Note: runtimes for tests are not linear, several factors can affect the runtime:

- network roundtrip time
- general ssh delays and retries
- repository delay (all EDB repository operations are serialized)
- load on the AWS hosts, influencing the virtual machine performance
- ssh jumphost delays (EDB is using a jumphost to access AWS infrastructure)

#### m7a.medium

- 2 segment hosts, 2 segments, 1st run: `8m0s`
- 2 segment hosts, 2 segments, 2nd run: `4m56s`
- 2 segment hosts, 4 segments, 1st run: `8m3s`
- 2 segment hosts, 4 segments, 2nd run: `5m7s`

- 4 segment hosts, 2 segments, 1st run: `9m58s`
- 4 segment hosts, 2 segments, 2nd run: `5m57s`
- 4 segment hosts, 4 segments, 1st run: `10m29s`
- 4 segment hosts, 4 segments, 2nd run: `5m55s`

- 10 segment hosts, 2 segments, 1st run: `17m51s`
- 10 segment hosts, 2 segments, 2nd run: `10m27s`
- 10 segment hosts, 4 segments, 1st run: `19m26s`
- 10 segment hosts, 4 segments, 2nd run: `10m19s`

- 20 segment hosts, 2 segments, 1st run: `28m53s`
- 20 segment hosts, 2 segments, 2nd run: `15m0s`
- 20 segment hosts, 4 segments, 1st run: `30m44s`
- 20 segment hosts, 4 segments, 2nd run: `15m5s`

4 GB RAM is not enough for the `coordinator` to manage 40 segments, operation times out.

In general, `m7a.medium` instances are too small to run 2 WarehousePG instances. Deployment will occasionally time out.

#### m7a.large

- 2 segment hosts, 2 segments, 1st run: `7m48s`
- 2 segment hosts, 2 segments, 2nd run: `5m0s`
- 2 segment hosts, 4 segments, 1st run: `7m49s`
- 2 segment hosts, 4 segments, 2nd run: `5m2s`

- 4 segment hosts, 2 segments, 1st run: `9m48s`
- 4 segment hosts, 2 segments, 2nd run: `5m48s`
- 4 segment hosts, 4 segments, 1st run: `9m17s`
- 4 segment hosts, 4 segments, 2nd run: `5m56s`

- 10 segment hosts, 2 segments, 1st run: `16m48s`
- 10 segment hosts, 2 segments, 2nd run: `9m30s`
- 10 segment hosts, 4 segments, 1st run: `17m21s`
- 10 segment hosts, 4 segments, 2nd run: `9m27s`

- 20 segment hosts, 2 segments, 1st run: `31m23s`
- 20 segment hosts, 2 segments, 2nd run: `15m8s`
- 20 segment hosts, 4 segments, 1st run: `30m54s`
- 20 segment hosts, 4 segments, 2nd run: `15m34s`

- 40 segment hosts, 2 segments, 1st run: `56m45s`
- 40 segment hosts, 2 segments, 2nd run: `27m31s`
- 40 segment hosts, 4 segments, 1st run: `50m40s`
- 40 segment hosts, 4 segments, 2nd run: `21m13s`

#### m7a.xlarge

- 2 segment hosts, 2 segments, 1st run: `7m12s`
- 2 segment hosts, 2 segments, 2nd run: `4m26s`
- 2 segment hosts, 4 segments, 1st run: `7m28s`
- 2 segment hosts, 4 segments, 2nd run: `4m30s`

- 4 segment hosts, 2 segments, 1st run: `8m10s`
- 4 segment hosts, 2 segments, 2nd run: `4m56s`
- 4 segment hosts, 4 segments, 1st run: `8m50s`
- 4 segment hosts, 4 segments, 2nd run: `4m58s`

- 10 segment hosts, 2 segments, 1st run: `15m3s`
- 10 segment hosts, 2 segments, 2nd run: `7m56s`
- 10 segment hosts, 4 segments, 1st run: `16m35s`
- 10 segment hosts, 4 segments, 2nd run: `7m58s`

- 20 segment hosts, 2 segments, 1st run: `26m40s`
- 20 segment hosts, 2 segments, 2nd run: `12m13s`
- 20 segment hosts, 4 segments, 1st run: `27m36s`
- 20 segment hosts, 4 segments, 2nd run: `13m0s`

- 40 segment hosts, 2 segments, 1st run: `50m52s`
- 40 segment hosts, 2 segments, 2nd run: `24m22s`
- 40 segment hosts, 4 segments, 1st run: `52m8s`
- 40 segment hosts, 4 segments, 2nd run: `21m20s`

#### m7a.2xlarge

- 2 segment hosts, 2 segments, 1st run: `7m3s`
- 2 segment hosts, 2 segments, 2nd run: `4m52s`
- 2 segment hosts, 4 segments, 1st run: `7m48s`
- 2 segment hosts, 4 segments, 2nd run: `4m32s`

- 4 segment hosts, 2 segments, 1st run: `8m32s`
- 4 segment hosts, 2 segments, 2nd run: `5m1s`
- 4 segment hosts, 4 segments, 1st run: `8m39s`
- 4 segment hosts, 4 segments, 2nd run: `4m59s`

- 10 segment hosts, 2 segments, 1st run: `15m17s`
- 10 segment hosts, 2 segments, 2nd run: `8m16s`
- 10 segment hosts, 4 segments, 1st run: `16m27s`
- 10 segment hosts, 4 segments, 2nd run: `8m5s`
- 10 segment hosts, 6 segments, 1st run: `17m38s`
- 10 segment hosts, 6 segments, 2nd run: `8m7s`
- 10 segment hosts, 8 segments, 1st run: `17m10s`
- 10 segment hosts, 8 segments, 2nd run: `8m6s`

- 20 segment hosts, 2 segments, 1st run: `27m21s`
- 20 segment hosts, 2 segments, 2nd run: `12m23s`
- 20 segment hosts, 4 segments, 1st run: `31m50s`
- 20 segment hosts, 4 segments, 2nd run: `16m46s`
- 20 segment hosts, 6 segments, 1st run: `29m34s`
- 20 segment hosts, 6 segments, 2nd run: `13m12s`
- 20 segment hosts, 8 segments, 1st run: `34m2s`
- 20 segment hosts, 8 segments, 2nd run: `15m46s`

- 40 segment hosts, 2 segments, 1st run: `62m13s`
- 40 segment hosts, 2 segments, 2nd run: `34m21s`
- 40 segment hosts, 4 segments, 1st run: `66m42s`
- 40 segment hosts, 4 segments, 2nd run: `38m54s`
- 40 segment hosts, 6 segments, 1st run: `65m25s`
- 40 segment hosts, 6 segments, 2nd run: `36m48s`
- 40 segment hosts, 8 segments, 1st run: `68m53s`
- 40 segment hosts, 8 segments, 2nd run: `33m59s`

##### Increase forks parameter

The following tests were run with an increased `forks` parameter, allowing for more parallel operations on the Ansible controller host (Apple M1 Max) while consuming more resources on the controller.

- 40 segment hosts, 8 segments, forks=21 (running 20 segments in parallel, plus coordinator), 1st run: `71m22s`
- 40 segment hosts, 8 segments, forks=21 (running 20 segments in parallel, plus coordinator), 2nd run: `37m33s`

- 40 segment hosts, 8 segments, forks=41 (running 40 segments in parallel, plus coordinator), 1st run: `72m40s`
- 40 segment hosts, 8 segments, forks=41 (running 40 segments in parallel, plus coordinator), 2nd run: `31m11s`
