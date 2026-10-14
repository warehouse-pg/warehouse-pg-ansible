# edb-repository

Installs the EDB package repository on Red Hat based systems. The WarehousePG, WEM, ClickHouse, OTel collector and ACP host agent packages all come from this repository, so the role must run before the roles that install them. The playbook runs it first in every play that installs EDB packages.

The role supports Red Hat based systems only.

## What the role does

1. Checks that the OS family is Red Hat and that `edb_token` is set and not empty.
2. Checks whether `/etc/yum.repos.d/enterprisedb-<edb_repository>.repo` already exists. If it does, the role stops here and changes nothing.
3. Otherwise:
   - downloads the repository setup script `https://downloads.enterprisedb.com/basic/<edb_repository>/setup.rpm.sh` to `/root/edb_setup.rpm.sh` (mode `0700`), authenticating with the token,
   - runs the script as root,
   - removes the script,
   - runs `yum makecache`.

Each download and execution step is retried up to 10 times, 5 seconds apart. The EDB repository denies too many parallel requests, so these steps are throttled with `serial_operations` (default: one host at a time). This might take a while if there are many segment hosts.

The role is idempotent. After the repository file exists, later runs only perform the file check.

## Run

The role has no tag of its own. It runs as part of the plays that need EDB packages, for example:

```bash
ansible-playbook -i <inventory> playbook.yml --tags deploy
```

## EDB token

The role needs an EDB token and a repository name. Ask your EDB contact for details.

| Variable | Default | Description |
|---|---|---|
| `edb_token` | `""` | EDB repository authentication token. Must be defined and non-empty, pre-flight fails otherwise. |
| `edb_repository` | `dev` | EDB repository name, for example `dev` or `gsupp`. |
| `serial_operations` | `1` | Throttle for concurrent download, install and refresh operations. |

The token is passed to the download as basic-auth password. The download task uses `no_log`, so the token does not appear in the Ansible output.

The playbook reads both values from local files on the Ansible controller, in a play tagged `always`:

```yaml
- name: Read EDB token from local files
  hosts: all
  gather_facts: false
  tags: always
  tasks:
    - name: Read token from local file
      ansible.builtin.set_fact:
        edb_token: "{{ lookup('file', lookup('env', 'HOME') ~ '/.edb-token') }}"
      run_once: true
      # hide the token
      no_log: true

- name: Read EDB repository from local files
  hosts: all
  gather_facts: false
  tags: always
  tasks:
    - name: Read repository from local file
      ansible.builtin.set_fact:
        edb_repository: "{{ lookup('file', lookup('env', 'HOME') ~ '/.edb-repository') }}"
      run_once: true
```

You don't have to follow this example. Set `edb_token` and `edb_repository` any other way in your inventory, `group_vars` or with `-e`, for example with Ansible Vault.

## Security

The role runs as root on the remote systems and requires Ansible `become` privileges (usually `sudo`).

It downloads a setup script from EDB and runs it as root. If you do not want to run arbitrary vendor scripts as root, exclude this role and set up the repository manually. Every role that installs EDB packages then only needs the repository to be configured on the host.

The script is written to `/root/edb_setup.rpm.sh` and removed after it ran. If the run fails between download and removal, the script stays on the host.
