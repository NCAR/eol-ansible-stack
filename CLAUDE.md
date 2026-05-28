# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

A Dockerized Ansible control node. All Ansible work runs **inside the container** — playbooks, `ansible-playbook`, and `ansible` ad-hoc commands are not meant to be run on the host directly.

The `../local/` directory (one level above the repo) holds everything that is **not** committed: SSH keys, vault secrets, bash history, and the real inventory. The repo itself contains playbooks and the Ansible configuration only.

## Container lifecycle

```bash
# Build and start
docker-compose up -d --build

# Shell into the container
docker-compose exec ansible-control bash          # as root
docker-compose exec -u ansible ansible-control bash  # as ansible user

# Mount an extra project directory at runtime
docker-compose run -u ansible -v /path/to/project:/ansible/playbooks/project ansible-control bash

# Stop
docker-compose down
```

## Running playbooks (from inside the container)

```bash
ansible-playbook playbooks/site.yml
ansible-playbook playbooks/deploy_pub_key.yml          # initial SSH key bootstrap
ansible-playbook playbooks/install_base.yml
ansible-playbook playbooks/update_installed.yml
ansible-playbook playbooks/node_info.yml               # saves JSON to local/files/
ansible-playbook playbooks/prometheus/deploy.yml       # targets [monitoring] group
```

Connectivity test:

```bash
ansible all -m ping
ansible all --list-hosts
```

## Architecture

### Directory layout

```
.
├── ansible/               # Ansible config + template playbooks
│   ├── ansible.cfg        # Primary config (inventory, vault, SSH, callbacks)
│   ├── filter_plugins/    # Custom Jinja2 filters for vault lookups
│   │   └── vault_filters.py
│   ├── inventory/         # Example/default inventory (not the real one)
│   ├── site.yml           # Example connectivity playbook
│   └── vault-manage.sh    # ansible-vault wrapper script
├── playbooks/             # Operational playbooks
│   ├── deploy_pub_key.yml
│   ├── install_base.yml
│   ├── node_info.yml
│   ├── site.yml
│   ├── update_installed.yml
│   └── prometheus/        # Prometheus+Grafana+Alertmanager stack
├── Dockerfile             # Ubuntu 22.04 + Python + Ansible
├── docker-compose.yml
└── docs/
```

### `../local/` volume (outside the repo)

```
../local/
├── inventory/
│   ├── inventory.ini              # Real host inventory
│   └── group_vars/all/
│       ├── all.yml
│       └── passwords.yml -> ../vault/passwords.yml  # symlink to vault
├── ssh-keys/
│   ├── ansible/id_ed25519{,.pub}  # mounted to /home/ansible/.ssh
│   └── root/id_ed25519{,.pub}     # mounted to /root/.ssh
└── vault/
    ├── passwords.yml              # ansible-vault encrypted secrets
    └── .vault_pass                # vault decryption key (chmod 600)
```

### Vault system

`local/vault/passwords.yml` is encrypted with `ansible-vault`. The vault password file path is `local/vault/.vault_pass` (configured in `ansible.cfg`).

Vault data structure:

```yaml
vault_passwords:
  "hostname-or-ip":
    username: "password"
vault_passwords_by_hostname:
  SHORTNAME:
    username: "password"
vault_ssh_key_passphrases:
  id_ed25519: "passphrase"
vault_service_credentials:
  smtp:
    username: "user"
    password: "pass"
```

#### Custom vault filter plugins (`ansible/filter_plugins/vault_filters.py`)

Three Jinja2 filters are available in all playbooks:

- `get_vault_password(vault_passwords, hostname, username, fallback_hostname=None)` — direct lookup
- `get_service_credential(vault_service_credentials, service, credential_type)` — service credential lookup
- `safe_password_lookup(vault_data, host_info, username=None)` — tries ansible_host → inventory_hostname → hostname, then falls back to `vault_passwords_by_hostname`

Typical usage in `group_vars/all/all.yml`:

```yaml
ansible_password: >-
  {{ vault_passwords | get_vault_password(
       ansible_host | default(inventory_hostname),
       ansible_user | default('root'),
       inventory_hostname) }}
```

## Vault management

```bash
./ansible/vault-manage.sh init          # create .vault_pass and encrypt passwords.yml
./ansible/vault-manage.sh edit          # edit encrypted vault
./ansible/vault-manage.sh view          # view without decrypting to disk
./ansible/vault-manage.sh encrypt       # encrypt plaintext vault file
./ansible/vault-manage.sh decrypt       # decrypt temporarily (re-encrypt when done)
./ansible/vault-manage.sh test          # verify vault access + run test playbook
./ansible/vault-manage.sh change-pass   # rekey vault
```

The script expects vault files at `ansible/vault/` (relative to script location). Inside the container this maps to `/ansible/vault/`.

## Bootstrapping a new project

`ansible/structure_init.sh` generates a role-based project skeleton from an inventory file. Roles are detected from `roles=` values in the inventory.

```bash
mkdir -p playbooks/my_project && cd playbooks/my_project
../../ansible/structure_init.sh ../../local/inventory/inventory.ini
```

## Prometheus playbook specifics

`playbooks/prometheus/deploy.yml` targets hosts in the `[monitoring]` inventory group. Key variables in `playbooks/prometheus/vars/main.yml`:

| Variable | Default | Notes |
|---|---|---|
| `prometheus_port` | `9090` | |
| `alertmanager_port` | `9093` | |
| `grafana_port` | `3000` | |
| `monitoring_groups` | `[webservers, databases, monitoring]` | groups included in node list |
| `prometheus_auto_start` | `false` | set to `true` to start containers automatically |

When `prometheus_auto_start` is false, start manually after the playbook runs:

```bash
cd ~/prometheus-stack && docker-compose up -d
```
