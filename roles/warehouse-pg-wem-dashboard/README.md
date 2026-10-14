# warehouse-pg-wem-dashboard

Installs the WarehousePG Enterprise Manager (WEM) - a monitoring and administration dashboard for WarehousePG. The role installs the `whpg-enterprise-manager` RPM, edits the RPM-shipped `/etc/edb/wem/wem.conf` in place, and enables the `wem` systemd service.

## Run

```bash
ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags warehouse-pg-wem-dashboard
```

## Where the role runs

The role targets the `wem_stack` inventory group. The host is selected by `wem_host`, set per cluster in the inventory (`cluster_<name>/vars`). The target host can be:

- A dedicated standalone host (no WHPG installed locally) - WEM connects to the WarehousePG cluster over the network.
- A WHPG coordinator.
- A WHPG standby.

The playbook runs the role on the `wem_stack` hosts.

## Prerequisites

- *EDB repository* configured on the target host. The role does not install it; on an air-gapped host the operator must arrange this manually. On a coordinator/standby it's already configured via `edb-repository`.

- The `acl` package, which the role installs itself (`tasks/00-acl.yml`); Ansible needs `setfacl` for `become_user` to an unprivileged user.

- *A WHPG database user* with superuser privileges. The role creates it itself (`whpg_user` / `whpg_password`), together with the matching `pg_hba.conf` entries, delegated to the coordinator. It manages only that one role; other roles stay with the `warehouse-pg` play via `whpg_db_roles` in the inventory. This step runs only when `wem_upgrade` is an install value. It is part of the `wem` and `warehouse-pg-wem-dashboard` tags, and can be run alone with `--tags warehouse-pg-db-roles` or `--tags update-pg-hba` (both select the same tasks). The `whpgroles` tag is a different one: it applies `whpg_db_roles` from the `warehouse-pg` role.

- The `warehouse-pg` role must be available on the roles path, because this role includes its tasks (`include_role`) for the database role setup.

> **Important:** when WEM is co-located with WarehousePG (coordinator or standby), `WHPG_UNIX_USER` and `WHPG_UNIX_GROUP` **must hold the same values in both plays**. Both roles create or chown files for that user, and a mismatch leaves the host with two different identities for the same files. Set the variables once in `group_vars` / inventory rather than per-play to guarantee this. The same applies to the `warehouse-pg` tasks this role includes for the DB role setup.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `whpg_host` | `127.0.0.1` | Hostname or IP of the WarehousePG cluster WEM monitors. If the coordinator is in the inventory, its IP is used automatically; otherwise this value (or `-e whpg_host=`) is used. |
| `whpg_port` | `5432` | WarehousePG coordinator port. |
| `whpg_user` | `wem_user` | **Required.** Database user WEM connects as. |
| `whpg_password` | `""` | **Required.** Password for `whpg_user`. Must be set by the operator, there is no built-in default. |
| `whpg_database` | `""` | WarehousePG database WEM connects to. Leave empty to use the default. |
| `wem_state_db_host` | `127.0.0.1` | **Required.** Host of the WEM state PostgreSQL instance. |
| `wem_state_db_port` | `5433` | WEM state database port. |
| `wem_state_db_user` | `wem_admin` | **Required.** PostgreSQL user for WEM state access. |
| `wem_state_db_password` | `""` | Password for `wem_state_db_user`. Required when `wem_state_db_host` is not on localhost. |
| `wem_state_db` | `wem` | WEM state database name. |
| `wem_port` | `8080` | Port the WEM web interface listens on. |
| `wem_admin_password` | `admin` | Initial admin password. Reset on first login. |
| `wem_cookie_secret` | *(auto-generated)* | Session cookie secret. Leave unset to have the RPM generate one on first run. |
| `wem_insecure_cookies` | `""` | Empty leaves `wem.conf` as it is: a new install gets secure cookies and an existing install keeps its current setting. `"1"` enables insecure cookies (plain HTTP). `"false"` (or `"0"`, `"no"`) empties it, giving secure cookies. |
| `wem_upgrade` | `install-but-no-automatic-upgrades` | Install/upgrade/remove policy. |

## Package state

| Value | Effect |
|---|---|
| `install-but-no-automatic-upgrades` (default) | Install if missing, don't auto-upgrade |
| `always-upgrade-to-latest-minor` | Upgrade to latest on every run |
| `absent` | Remove the RPM |

## Other knobs

See `defaults/main.yml` for the full list including: PXF integration, and service control.
