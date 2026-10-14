# warehouse-pg

Deploys and configures a WarehousePG cluster on the hosts of a `cluster_<cluster_name>` inventory group. This is the main role of the collection: it prepares the operating system, installs the packages, initializes the cluster and keeps it in line with the inventory (standby, resource management, extensions, database roles, expansion).

The role:

- validates the inventory variables (pre-flight),
- creates the `WHPG_UNIX_USER` / `WHPG_UNIX_GROUP` account (default `gpadmin`) with a locked password,
- writes the cluster hostfiles into the home directory of that user and updates `/etc/hosts` on all cluster hosts,
- prepares the OS (for example cgroups for resource groups),
- exchanges SSH keys between all cluster hosts,
- installs the WarehousePG packages (and optionally `whpg-backup`),
- initializes the database with `gpinitsystem` and applies `pg_hba.conf` customizations (coordinator only),
- reconciles the standby coordinator and the resource groups or queues,
- installs or removes the optional extension packages,
- creates or updates database roles from `whpg_db_roles`,
- expands the cluster when new segment hosts are added.

The role supports Red Hat based systems only, and must run as root, so Ansible `become` is required.

## Run

The role is called from `playbook.yml` in this example:

```bash
ansible-playbook -i <inventory> playbook.yml --tags deploy
```

Only the database roles:

```bash
ansible-playbook -i <inventory> playbook.yml --tags whpgroles
```

Only the `pg_hba.conf` entries and the WEM database role (needs a `wem_stack` group in the inventory):

```bash
ansible-playbook -i <inventory> playbook.yml --tags update-pg-hba
```

## Task flow

| File | What it does |
|---|---|
| `00-acl.yml` | Installs the `acl` package, required for `become_user` to an unprivileged user. |
| `01-pre-flight-checks.yml` | Validates all required and optional variables and checks that privilege escalation works. |
| `02-unix-user.yml` | Creates the Unix user and group. |
| `03-hostfiles.yml` | Writes the per-cluster hostfiles. Adds or removes the standby hostfile depending on the inventory. |
| `04-os.yml` | Updates `/etc/hosts` and prepares the OS (cgroups for resource groups). |
| `05-ssh-keyexchange.yml` | Exchanges SSH public keys and host keys across the cluster, then tests passwordless login from coordinator and standby. |
| `06-install.yml` | Installs the packages and creates the data directories. |
| `07-initialize-database.yml` | Initializes the database, applies `pg_hba.conf` customizations, reconciles the standby, syncs resource groups or queues (coordinator only). |
| `08-extensions.yml` | Installs or removes the extension packages. |
| `10-roles.yml` | Creates or updates database roles from `whpg_db_roles` (tag `whpgroles`). |
| `09-expand-cluster.yml` | Expands the cluster when `whpg_expand_db` is `true` (coordinator only). |

`11-update-pg-hba.yml` is not part of this flow. It is a standalone task file that updates `pg_hba.conf` on the coordinator. The `warehouse-pg-wem-dashboard` role includes it, and it can be run with `--tags update-pg-hba`.

## Prerequisites

- *EDB repository* configured on the target hosts (done by `edb-repository` in the playbook), unless `whpg_install_packages` is `false` and the packages are installed manually.
- `whpg_data_directory` must be set and the directory must already exist on every host. The role default is empty and is rejected.
- Each host needs `whpg_role` (`coordinator`, `standby` or `segment`), `hostname` and `ip_address`.
- Each cluster group needs `cluster_name` and `whpg_cluster_name`.
- Ansible `become` (usually `sudo`) must work for the connecting user. Pre-flight fails otherwise.
- With `whpg_resource_management: group-v2`, every host must run cgroup v2 (`/sys/fs/cgroup/cgroup.controllers` must exist).

## Key variables

The full list is in the main [README](../../README.md#role-warehouse-pg). The most important ones:

| Variable | Default | Description |
|---|---|---|
| `whpg_data_directory` | `""` _(required)_ | Base data directory. Must exist on every host. |
| `whpg_role` | _(none - required)_ | Role of the host: `coordinator`, `standby` or `segment`. |
| `WHPG_UNIX_USER` | `gpadmin` | Unix user that owns the installation. |
| `WHPG_UNIX_GROUP` | `gpadmin` | Unix group of that user. |
| `whpg_major_version` | `7` | WarehousePG major version, `6` or `7`. |
| `whpg_seg_num` | `2` | Primary segments per host. Only evaluated at cluster creation. |
| `whpg_port` | `5432` | Coordinator port. |
| `whpg_seg_base_port` | `6000` | Base port for primary segments. |
| `whpg_seg_enable_mirrors` | `false` | Create mirror segments. |
| `whpg_seg_mirror_base_port` | `7000` | Base port for mirror segments. |
| `whpg_upgrade` | `install-but-no-automatic-upgrades` | Package policy, see below. |
| `whpg_install_packages` | `true` | When `false`, the packages must be installed manually. |
| `whpg_install_backup` | `true` | Install the `whpg-backup` package. |
| `whpg_resource_management` | `group` | `queue`, `group` or `group-v2`. |
| `whpg_resource_groups` / `whpg_resource_queues` | `[]` | Resource group or queue definitions. |
| `whpg_db_roles` | `[]` | Database roles to create or update. |
| `additional_pg_hba_conf` | _(none)_ | Extra `pg_hba.conf` entries for the coordinator. |
| `whpg_extension_<name>` | `false` | Install (`true`), remove (`false`) or install a specific RPM (string). |
| `whpg_expand_db` | `true` | Expand the cluster automatically when segment hosts are added. |
| `greenplum_path` | `/usr/edb/whpg7/greenplum_path.sh` | Environment script sourced before the WarehousePG tools. |
| `DNF_TIMEOUT` | `300` | Maximum time (seconds) to wait for the RPM repository. |
| `serial_operations` | `1` | Throttle for concurrent package installs. |

## Package state

| Value of `whpg_upgrade` | Effect |
|---|---|
| `install-but-no-automatic-upgrades` (default) | Install if missing, don't auto-upgrade |
| `always-upgrade-to-latest-minor` | Upgrade to latest on every run |

```bash
ansible-playbook -i <inventory> playbook.yml -e whpg_upgrade=always-upgrade-to-latest-minor --tags deploy
```

## Database roles

`whpg_db_roles` creates a login role if it does not exist, or updates the password and privileges if it does. It runs on the coordinator only. The names `postgres` and `gpadmin` are reserved and rejected.

```yaml
whpg_db_roles:
  - name: reporting
    password: "<password>"
  - name: etl_admin
    password: "<password>"
    createdb: true
```

| Key | Required | Default | Description |
|---|---|---|---|
| `name` | yes | - | Role name. |
| `password` | no | - | Login password. Omit for passwordless roles. |
| `superuser` | no | `false` | Grant `SUPERUSER`. |
| `createdb` | no | `false` | Grant `CREATEDB`. |
| `createrole` | no | `false` | Grant `CREATEROLE`. |

Keep the passwords in Ansible Vault or another secret store, not in plain text in the inventory.

## Customizing `pg_hba.conf`

Two mechanisms extend the coordinator's `pg_hba.conf`. Both can be used together, and the database is reloaded when either changes the file.

- A file `templates/pg_hba_<cluster_name>.conf` in this role, processed through Jinja2 and added in an Ansible-managed block.
- The `additional_pg_hba_conf` variable, a list of entries with `conntype`, `database`, `user`, `method` and optionally `address` and `options`.

The entry schema and an example are in the main [README](../../README.md#customizing-pg_hbaconf).

The template `templates/pg_hba_monitoring_wem.conf` is applied when the inventory has a `wem_stack` group. It allows `whpg_user` from `127.0.0.1` and `gpadmin` and `whpg_user` from the WEM host, with `md5` authentication.

## Standby coordinator

Any host with `whpg_role: standby` is the desired standby, at most one per cluster. On each run the role compares the inventory with `gpstate -f` and adds or removes the standby with `gpinitstandby`. Changing the standby host in place is rejected. Remove the old standby first, run the playbook, then set the new one. See the main [README](../../README.md#standby-coordinator).

## Workload management

`whpg_resource_management` selects Resource Queues (`queue`) or Resource Groups (`group`, `group-v2`). The value maps to the `gp_resource_manager` GUC. When the running database uses a different manager, the role switches it with `gpconfig` and restarts the cluster with `gpstop -r`. See the main [README](../../README.md#workload-management) for the definition format and sync behavior.

## Extensions

Each `whpg_extension_<name>` variable installs (`true`), removes (`false`) or installs a specific RPM (string with an NVR, NEVRA or a path to an `.rpm`). Supported names: `diskquota`, `fts_enhanced`, `hll`, `madlib`, `madlib1`, `pgfs`, `pgvector`, `plr`, `postgis`, `pxf`, `q3c`, `user_profile`, `vectorchord`, `whpg_fdw`. See the main [README](../../README.md#extensions) for examples.

## Generated files

In the home directory of `WHPG_UNIX_USER`:

```
cluster_hostfile_all_<cluster_name>
cluster_hostfile_coordinator_<cluster_name>
cluster_hostfile_standby_<cluster_name>      # only if a standby host is defined
cluster_hostfile_segments_<cluster_name>     # only if segment hosts are defined
gpinitsystem_config_<cluster_name>
```

## Security notes

- `WHPG_UNIX_USER` is created with a locked password (`!`), so it can't log in with a password.
- The SSH key exchange authorizes every public key found in `~/.ssh/*.pub` of every cluster host for `WHPG_UNIX_USER` on all hosts. Anyone who controls one host as that user can log in to all other cluster hosts. Keep `~/.ssh` free of other `*.pub` files.
- Passwords from `whpg_db_roles` are passed on the `psql` command line and can be visible in the process list and the database logs while the task runs. Task output is hidden with `no_log`.
- `additional_pg_hba_conf` accepts any authentication method, including `trust`, and any address. Review the entries before applying them.
- The WEM `pg_hba.conf` entries use `md5`, and allow `gpadmin` from the WEM host.
