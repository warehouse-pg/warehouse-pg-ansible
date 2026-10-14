# acp-host-agent

Installs the `edb-acp-host-agent` RPM from the EDB repository on **all cluster nodes** (`whpg_clusters`: coordinator, standby and segments). The agent connects to the WEM relay port and registers activity handlers that WEM uses to execute cluster management operations remotely.

The agent's configuration file `/etc/edb/acp-host-agent/acp-host-agent.conf` is edited in place - only the keys controlled by Ansible variables are written; all RPM-shipped commented examples remain untouched.

## Run

```bash
ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags acp-host-agent
```

## Key variables

| Variable | Default | Description |
|---|---|---|
| `acp_agent_package_name` | `edb-acp-host-agent` | RPM package name from the EDB repository. |
| `acp_agent_version` | `""` | Pin a specific RPM version (e.g. `1.2.3`). Leave empty to install whatever the EDB repository provides. |
| `acp_agent_upgrade` | `install-but-no-automatic-upgrades` | Install/upgrade/remove policy. Accepted values: `install-but-no-automatic-upgrades`, `always-upgrade-to-latest-minor`, `absent`. |
| `wem_connect_address` | `{{ wem_connect_host }}:{{ wem_port + 1 }}` | WEM relay address (`host:port`). Derived in the role defaults from `wem_host` IP (`wem_connect_host`, fallback `127.0.0.1`) and `wem_port` (fallback `8080`) + 1. Override via inventory or `-e wem_connect_address=<host>:<port>`. |
| `wem_task_queues` | auto-derived from `whpg_role` | Task queue(s) the agent polls. Comma-separated. Defaults to `coordinator-tasks` for coordinators, `standby-tasks` for standbys. For segments no value is written to the agent config and the agent uses its built-in default. |
| `agent_id` | `""` | Stable per-host identifier. Leave empty to auto-derive from DMI UUID / MAC. |
| `agent_state_dir` | `""` | Directory where the Ed25519 enrollment keypair is stored. Leave empty to use the RPM default. |
| `wem_database` | `""` | WHPG database psql connects to for WEM data (default: `wem` if unset). |
| `agent_log_level` | `info` | Log level filter (`error`, `warn`, `info`, `debug`, `trace`). |
| `greenplum_path` | `/usr/edb/whpg7/greenplum_path.sh` | Path to `greenplum_path.sh`, sourced before running WHPG commands. |

## Notes

- Runs on all hosts in `whpg_clusters` (coordinator, standby and segments).
- `wem_connect_address` is resolved automatically from `wem_host` in the role defaults - it points to the WEM host's private IP at `wem_port + 1`. Override via inventory or `-e wem_connect_address=<host>:<port>`.
- WEM runs as a single process; no additional services need to be managed separately.
