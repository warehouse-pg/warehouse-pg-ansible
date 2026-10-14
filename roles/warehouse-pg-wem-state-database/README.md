# warehouse-pg-wem-state-database

Installs PostgreSQL 15 on the `wem_stack` host via the DNF AppStream module. After installation, the role creates the WEM state database and user, and configures `pg_hba.conf` so host connections use password authentication. Loopback (`127.0.0.1`, `::1`) follows `wem_state_db_local_trust`.

Runs on port `5433` by default to avoid conflicting with WHPG on port `5432`.

## Run

```bash
ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags warehouse-pg-wem-state-database
```

## Key variables

| Variable | Default | Description |
|---|---|---|
| `pg_version` | `15` | PostgreSQL major version to install via DNF AppStream. |
| `pg_port` | `5433` | Port PostgreSQL listens on. Uses 5433 to avoid conflict with WHPG on 5432. |
| `pg_data_dir` | `/var/lib/pgsql/data` | PostgreSQL data directory. |
| `pg_service` | `postgresql` | systemd service name. |
| `wem_state_db` | `wem` | Name of the WEM state database to create. |
| `wem_state_db_user` | `wem_admin` | PostgreSQL role to create for WEM state access. |
| `wem_state_db_password` | `""` | Password for `wem_state_db_user`. Required unless trust auth is in effect for loopback. |
| `wem_state_db_local_trust` | `""` | Loopback authentication. Empty leaves `pg_hba.conf` as it is: a new install uses password auth and an existing install keeps what it has, including trust. `true` sets trust on `127.0.0.1` and `::1` (empty password allowed; the role prints a warning). `false` requires passwords and replaces trust. |
