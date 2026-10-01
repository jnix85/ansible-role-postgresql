# PostgreSQL Ansible Role (`ansible-role-postgresql`)

Comprehensive, production-ready Ansible role to install, configure, harden, and manage **PostgreSQL** on Debian and Ubuntu systems.

## Features

- **Official PGDG Repositories**: Automatically configures the official PostgreSQL Global Development Group (`apt.postgresql.org`) repository with signed keyrings.
- **Configurable PostgreSQL Version**: Defaults to PostgreSQL 16; seamlessly supports 14, 15, and 17.
- **Clean Configuration Architecture**: PostgreSQL settings are managed via drop-in configuration (`conf.d/99-ansible-custom.conf`), keeping base vendor configuration intact.
- **Client Authentication (HBA)**: Declarative, templated `pg_hba.conf` and `pg_ident.conf` supporting modern `scram-sha-256` authentication and CIDR subnet restrictions.
- **Declarative Provisioning**: Automated creation and management of databases, users, passwords, and permissions via the `community.postgresql` collection.
- **Extension Management**: Automatic extension package installation and database activation (`pgvector`, `postgis`, `pg_stat_statements`, `pgaudit`, etc.).
- **Physical Streaming Replication**: Primary/standby streaming replication support with replication slots and automated `pg_basebackup` standby bootstrapping.
- **Automated Backup System**: Systemd service & timer for scheduled database dumps (compressed with `gzip`), SHA256 verification checksums, and configurable retention pruning.
- **PgBouncer Connection Pooling**: Optional integrated PgBouncer pooler setup with transaction pooling, authentication lists, and service management.
- **Prometheus Monitoring**: Optional Prometheus `postgres_exporter` service setup with `pg_monitor` grants.

---

## Supported Operating Systems

- **Debian**: 12 (Bookworm), 13 (Trixie)
- **Ubuntu**: 22.04 (Jammy), 24.04 (Noble)

---

## Requirements

Ansible collections required (defined in `requirements.yml`):
- `community.postgresql >= 3.4.0`
- `community.general >= 10.0.0`

Install requirements:
```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Quick Start

### 1. Configure Inventory

Edit `inventory/hosts.yml`:
```yaml
all:
  children:
    postgresql_servers:
      children:
        postgresql_primary:
          hosts:
            pg1.clawduino.com:
              ansible_host: 10.1.0.60
```

### 2. Configure Variables

Set variables in `inventory/group_vars/postgresql_servers/main.yml` or host_vars:
```yaml
postgresql_version: 16
postgresql_listen_addresses: "*"
postgresql_shared_buffers: "2GB"

postgresql_hba_custom_rules:
  - type: host
    database: all
    user: all
    address: "10.1.0.0/23"
    auth_method: scram-sha-256

postgresql_databases:
  - name: authentik
    owner: authentik
  - name: netbox
    owner: netbox

postgresql_users:
  - name: authentik
    password: "{{ vault_authentik_db_password }}"
    db: authentik
    priv: "ALL"
  - name: netbox
    password: "{{ vault_netbox_db_password }}"
    db: netbox
    priv: "ALL"
```

### 3. Deploy

```bash
# Check syntax
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --syntax-check

# Dry-run
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff

# Run full deployment
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

---

## Playbooks

| Playbook | Purpose |
|---|---|
| `playbooks/site.yml` | Full end-to-end installation, configuration, user/database provisioning, backups, and optional components. |
| `playbooks/deploy.yml` | Fast execution updating configuration, databases, users, and extensions without re-running package manager tasks. |
| `playbooks/backup.yml` | Triggers an immediate manual backup on target hosts and outputs the status. |

---

## Role Variables

### Feature Toggles

| Variable | Default | Description |
|---|---|---|
| `postgresql_install_packages` | `true` | Whether to install PostgreSQL packages. |
| `postgresql_manage_service` | `true` | Whether to ensure systemd service state. |
| `postgresql_manage_configuration` | `true` | Whether to deploy configuration and HBA files. |
| `postgresql_manage_replication` | `false` | Enable primary/standby streaming replication. |
| `postgresql_manage_databases` | `true` | Manage declarative database lists. |
| `postgresql_manage_users` | `true` | Manage declarative user and permission lists. |
| `postgresql_manage_extensions` | `true` | Manage PostgreSQL extensions. |
| `postgresql_manage_backup` | `true` | Deploy automated systemd backup service & timer. |
| `postgresql_exporter_enabled` | `false` | Install and configure Prometheus `postgres_exporter`. |
| `postgresql_pgbouncer_enabled` | `false` | Install and configure PgBouncer connection pooler. |

### PostgreSQL Configuration

| Variable | Default | Description |
|---|---|---|
| `postgresql_version` | `16` | Major version of PostgreSQL to install. |
| `postgresql_listen_addresses` | `"localhost"` | IP addresses to listen on (`"*"` for all). |
| `postgresql_port` | `5432` | Port to listen on. |
| `postgresql_shared_buffers` | `"256MB"` | Memory allocated to shared memory buffers. |
| `postgresql_work_mem` | `"8MB"` | Memory used by internal sort operations and hash tables. |
| `postgresql_maintenance_work_mem` | `"64MB"` | Memory used by maintenance operations (VACUUM, CREATE INDEX). |
| `postgresql_effective_cache_size` | `"1GB"` | Planner's assumption about the effective size of disk cache. |
| `postgresql_max_connections` | `100` | Maximum number of concurrent connections. |
| `postgresql_password_encryption` | `"scram-sha-256"` | Hashing algorithm for passwords. |
| `postgresql_custom_conf_parameters` | `{}` | Key-value dictionary of extra custom settings in `99-ansible-custom.conf`. |

### Automated Backups

| Variable | Default | Description |
|---|---|---|
| `postgresql_backup_dir` | `"/var/backups/postgresql"` | Local directory for storing database backups. |
| `postgresql_backup_type` | `"individual"` | `"individual"` (separate dump per database) or `"all"` (`pg_dumpall`). |
| `postgresql_backup_retention_days` | `7` | Days to retain backup archives before pruning. |
| `postgresql_backup_calendar` | `"*-*-* 02:30:00"` | Systemd OnCalendar schedule expression. |

---

## Streaming Replication Guide

To deploy primary and standby nodes:

1. In `inventory/hosts.yml`, configure hosts under `postgresql_primary` and `postgresql_standby`.
2. Define replication settings in `group_vars`:
```yaml
postgresql_manage_replication: true
postgresql_replication_user: "replicator"
postgresql_replication_password: "{{ vault_postgresql_replication_password }}"
postgresql_replication_slot_name: "standby1_slot"
```
3. For primary hosts (`postgresql_replication_role: "primary"`):
   - Creates the `replicator` role with `REPLICATION` privilege.
   - Creates the physical replication slot.
   - Adds the `replication` line in `pg_hba.conf`.
4. For standby hosts (`postgresql_replication_role: "standby"`):
   - Sets `postgresql_replication_primary_host: "10.1.0.60"`.
   - Bootstraps the data directory via `pg_basebackup` with `-R` (generating `standby.signal`).

---

## Restoring from Backup

Backups created by the automated service are stored in `/var/backups/postgresql/` as compressed `.sql.gz` files alongside their `.sha256` checksums.

### Restoring an individual database:
```bash
# Verify checksum
sha256sum -c /var/backups/postgresql/pg_mydb_20260911_023000.sql.gz.sha256

# Restore into target database
gunzip -c /var/backups/postgresql/pg_mydb_20260911_023000.sql.gz | sudo -u postgres psql -d mydb
```

---

## License

MIT
