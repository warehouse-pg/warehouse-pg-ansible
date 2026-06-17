# WarehousePG Ansible Roles

Ansible roles for deploying and managing WarehousePG clusters.

## Roles

### edb-repository

Installs the EDB package repository on Red Hat based systems. This role must run before `warehouse-pg`, in order to make the WarehousePG packages available.

#### EDB Token

This role requires an EDB token and a repository name.

Set the variable `$edb_token` with the token, and the variable `$edb_repository` with the repository name.

The repository name defaults to `dev`, can also be `gsupp` or another available repository.
Ask your EDB contact for details.

#### Installation process

This role must run as root, and therefore requires Ansible `become` privileges (usually `sudo`). It will download a script which sets up the repository.

If you do not want to run arbitrary scripts (`setup.sh`) as root, exclude this role and setup the repository manually.

### warehouse-pg

Deploys and configures a WarehousePG cluster. Handles pre-flight validation, OS-level user/group setup, SSH key exchange, package installation, data directory calculation, and cluster hostfile generation.

**Task flow:**

1. `pre-flight-checks.yml`: Validates all required and optional variables
2. `unix-user.yml`: Creates the user and group specified in `$WHPG_UNIX_USER` and `$WHPG_UNIX_GROUP` (usually `gpadmin:gpadmin`)
3. `hostfiles.yml`: Generates per-cluster hostfiles in the Unix user's home directory
4. `os.yml`: Updates the `/etc/hosts` files on all cluster systems
5. `ssh-keyexchange.yml`: Exchanges SSH keys across the cluster
6. `install.yml`: Installs WarehousePG and optional backup packages, calculates data directories
7. `initialize-database.yml`: Initializes the WarehousePG database

### warehouse-pg-cluster-data

Snapshots the state of every cluster declared as a child of `whpg_clusters`. Runs on each cluster's coordinator (`whpg_role: coordinator`); other hosts short-circuit out. Captures the database version, global cluster localization, connection / FTS / interconnect / network tuning / memory GUCs, OS-level memory tuning, segment topology, resource manager configuration (with the active groups or queues exported), non-default GUCs, tablespaces with per-segment locations, all databases, available and installed extensions, the coordinator's `pg_hba.conf` (comment lines stripped), and an inventory of the cluster hostfiles on disk.

Writes two files into the Unix user's home directory on each coordinator:

```
~/cluster_state_<cluster_name>.txt   # human-readable
~/cluster_state_<cluster_name>.json  # machine-readable
```

Invoked as the last play of `playbook.yml`, or on its own via `ansible-playbook cluster-data.yml`.

> **Note:** This role is **not idempotent**. It rewrites both `cluster_state_<cluster_name>.{txt,json}` files on every run; the `scanned_at` timestamp changes each time, and any drift in the cluster state will surface in the file contents. Expect every Ansible run to report this role as `changed`.

## Inventory Example

```yaml
whpg_clusters:
  children:
    # --- Single-node cluster ---
    cluster_alpha:
      vars:
        cluster_name: "alpha"
        whpg_data_directory: "/data/whpg"
        whpg_cluster_name: "alpha"
      hosts:
        node-single:
          whpg_role: coordinator

    # --- Multi-node cluster with standby ---
    cluster_beta:
      vars:
        cluster_name: "beta"
        whpg_data_directory: "/data/whpg"
        whpg_cluster_name: "beta"
      hosts:
        node-x72j:
          whpg_role: coordinator
        node-a91p:
          whpg_role: standby
        node-z110:
          whpg_role: segment
        node-q442:
          whpg_role: segment
        node-r993:
          whpg_role: segment

    # --- Two-node cluster ---
    cluster_gamma:
      vars:
        cluster_name: "gamma"
        whpg_data_directory: "/data/whpg"
        whpg_cluster_name: "gamma"
      hosts:
        srv-prod-01:
          whpg_role: coordinator
        srv-prod-02:
          whpg_role: segment
```

All clusters are defined as children of `whpg_clusters`. Each host has a `whpg_role` variable set to one of: `coordinator`, `standby`, or `segment`.

A so-called "single node" installation only uses a server with `whpg_clusters=coordinator`, and no segment servers. The role will figure out that this is a single node installation, and adapt the configuration accordingly.

## Usage

Create an Ansible inventory which defines the roles for each server. See the example `Inventory` section earlier in this document.

Create a Playbook which applies the roles on the servers. Example:

```
---

- name: Deploy WarehousePG on all clusters
  hosts: whpg_clusters
  become: true
  roles:
    - role: edb-repository
    - role: warehouse-pg
```

## Deploy cluster(s) using Ansible

Run from inside this directory.

For installing WarehousePG, use:

```
ansible-playbook playbook.yml
```

## Generated files

The role will generate a number of files In the home directory of `WHPG_UNIX_USER`:

```
~/cluster_hostfile_all_<cluster_name>            # contains all hostnames
~/cluster_hostfile_coordinator_<cluster_name>.   # contains the hostname of the coordinator
~/cluster_hostfile_standby_<cluster_name>        # only if standby host is defined
~/cluster_hostfile_segments_<cluster_name>       # only if segment hosts are defined
~/gpinitsystem_<cluster_name>                    # the gpinitsystem file used to initialize the cluster
```

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
| `hostX` | `hostY` | fail — changing the standby hostname in place is not allowed |

### Data directory layout

The standby's data directory lives under `whpg_standby_data_directory` (default `<whpg_data_directory>/v<major_version>/standby`), with the standby data placed at `<whpg_standby_data_directory>/<seg_prefix>-1` (e.g. `/data/whpg/v7/standby/gpseg-1`). `gpinitstandby` is invoked with `-S` pointing at that path, so the standby host does **not** need the coordinator's directory path (e.g. `/data/whpg/v7/coordinator`) to exist. The parent (`whpg_standby_data_directory`) is created on the standby host by `06-install.yml`.

### Swapping the standby

To move the standby to a different host, change the inventory in two steps:

1. Remove `whpg_role: standby` from the current standby host and run the playbook. The old standby is removed via `gpinitstandby -r`.
2. Set `whpg_role: standby` on the new host and run the playbook again. The new standby is created via `gpinitstandby -s`.

Attempting both in a single inventory change (changing which host carries `whpg_role: standby` while the old standby is still configured in the database) is rejected with an explicit failure, since `gpinitstandby` does not support an in-place hostname change.

## Workload management

WarehousePG supports two mutually exclusive workload managers, selected via the variable `whpg_resource_management`. Valid values:

- `queue` — Resource Queues. Enforces concurrency, memory, and priority inside the database. No OS-level prep required.
- `group` — Resource Groups, with **auto-detection** of cgroup v1 vs v2 based on what is mounted on the host.
- `group-v2` — Resource Groups, **force cgroup v2**. Pre-flight rejects this on hosts that do not have `/sys/fs/cgroup/cgroup.controllers` (i.e. that aren't running cgroup v2 unified).

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
4. `ALTER` an existing entry only when at least one attribute differs from inventory — no statements are sent if everything already matches (keeps the audit log clean).
5. `DROP` any database entry not present in inventory, skipping the built-in defaults listed above.

Leaving the relevant list (`whpg_resource_groups` or `whpg_resource_queues`) at its empty default is supported. No entries will be created or updated, but the cleanup step will still drop any custom (non-default) entries that exist in the database.

See `roles/warehouse-pg/defaults/main.yml` for fully commented examples.

## Extensions

The `warehouse-pg` role can install optional WarehousePG extensions. Each extension has an associated `whpg_extension_<name>` variable that defaults to `false`. Set the variable to `true` to install the extension's package. The role automatically selects the package matching `$whpg_major_version`; extensions that are not available for the configured major version are skipped.

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
| `whpg_data_directory` | group/host | Base data directory path. Must exist as a directory on every host. |
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
| `whpg_seg_base_port` | `6000` | Base port for primary segments (written to `PORT_BASE` in `gpinitsystem` config). |
| `whpg_upgrade` | `"install-but-no-automatic-upgrades"` | Upgrade policy for warehouse-pg packages. The default `"install-but-no-automatic-upgrades"` is the safe setting: it installs the packages if missing but will not upgrade an already-installed version (maps to the `dnf` state `present`). `"always-upgrade-to-latest-minor"` auto-upgrades to the latest available minor version on every run (maps to the `dnf` state `latest`). |
| `whpg_use_versioned_directories` | `true` | When `true`, data directories include a `v<major_version>/` subdirectory. |
| `whpg_seg_enable_mirrors` | `true` | Enable mirror segments in the `gpinitsystem` configuration. |
| `whpg_seg_mirror_base_port` | `7000` | Base port for mirror segments. |
| `whpg_default_database` | `""` | Default database name. Must be a valid PostgreSQL identifier (1--63 chars, starts with letter or underscore) when non-empty. |
| `whpg_install_packages` | `true` | Whether the role should install packages. When `false`, the user is expected to install them. |
| `whpg_install_backup` | `false` | Set to `true` to install the `whpg-backup` package. |
| `whpg_resource_management` | `"group"` | Workload manager: `queue`, `group`, or `group-v2`. Maps to the `gp_resource_manager` GUC. See the "Workload management" section. |
| `whpg_resource_groups` | `[]` | List of resource group definitions applied when `whpg_resource_management` is `group` or `group-v2`. Empty by default. |
| `whpg_resource_queues` | `[]` | List of resource queue definitions applied when `whpg_resource_management` is `queue`. Empty by default. |
| `DNF_TIMEOUT` | `120` | Maximum time (seconds) to wait for the RPM repository. |
| `whpg_remove_old_gpAdminLogs` | `false` | Add maintenance jobs to remove old `gpAdminLogs` logfiles on coordinator/standby. |
| `whpg_remove_old_logs` | `false` | Add maintenance job to remove old data-directory logfiles. |
| `greenplum_path` | `"/usr/local/greenplum-db/greenplum_path.sh"` | Path to the `greenplum_path.sh` environment script. |

#### Optional variables (no default; validated only when defined)

| Variable | Description |
|---|---|
| `whpg_seg_base_port` | Base port for segments. Integer, 1024--65535. Validated only when defined and non-empty. |
| `additional_pg_hba_conf` | List of additional `pg_hba.conf` entries to apply on the coordinator. See the "Customizing `pg_hba.conf`" section above for the entry schema. |

#### Running upgrades

The `whpg_upgrade` variable defaults to `install-but-no-automatic-upgrades`, which is a safe choice. The playbook can run again, without package changes applied and potentially forcing database restarts.

This variable can be overridden on the command line. Example:

```
ansible-playbook playbook.yml -e "whpg_upgrade=always-upgrade-to-latest-minor"
```

This will apply the new setting only for this run of the playbook, making it safe to apply potential updates when convenient.
